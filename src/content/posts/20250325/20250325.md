---
title: Introduce to gRPC-Gateway
published: 2025-03-25
description: "gRPC-Gateway provides a service that generates a reverse-proxy server which translates a RESTful HTTP API into gRPC with simple configuration."
image: ""
tags: [Go, gRPC]
category: "Go"
draft: false
lang: ""
---

## Background

Recently, I wanted to integrate gRPC into my project. However, since some browsers and external services might not fully support the HTTP/2 protocol, I began designing the system to be compatible with both HTTP/1.1 (RESTful APIs) and HTTP/2 (gRPC). That's when I came across gRPC-Gateway — a powerful tool that bridges the gap between RESTful HTTP APIs and gRPC services.

::github{repo="grpc-ecosystem/grpc-gateway"}

The example code in this article is based on the following project :

::github{repo="nu1lspaxe/finance-manager-v2"}

**gRPC-Gateway** provides a powerful way to extend your existing `.proto` definitions by adding HTTP annotations. Using these annotations, the tool automatically generates an HTTP reverse proxy that translates RESTful HTTP/1.1 calls into gRPC requests.

Before using gRPC-Gateway, compiling a `<service>.proto` file typically generates two files :

- `<service>.pb.go`, which contains Go structs and interfaces for the Protocol Buffers messages defined in the `.proto` file, along with serialization and deserialization logic
- `<service>_grpc.pb.go`, which contains the gRPC-specific server and client interfaces, including the service definitions and method stubs for the gRPC service

After applying gRPC-Gateway, an additional file is generated :

- `<service>.pb.gw.go`, which contains the HTTP proxy logic based on your annotations

:::warning
Some features of gRPC-Gateway are not yet fully supported. For a detailed list of features, please refer to the [document](https://github.com/grpc-ecosystem/grpc-gateway?tab=readme-ov-file#features).
:::

## Preface

Readers of the following article are assumed to have a basic understanding of Golang syntax and the gRPC protocol. Therefore, the focus will be on explaining how gRPC-Gateway implements HTTP proxying, without going into too much detail about the gRPC protocol itself.

This article assumes that readers have a basic understanding of Golang syntax and the gRPC protocol. As such, the focus will be on how gRPC-Gateway enables HTTP proxying, rather than covering the fundamentals of gRPC itself.

## Code Preview

### `users/proto/users.pb.gw.go`

From the code snippet, we can see that `RegisterUserServiceHandlerFromEndpoint` connects a gRPC service client endpint to an HTTP server implemented using `*runtime.ServeMux`. This effectively exposes a RESTful interface, allowing clients to access the gRPC service over the HTTP protocol.

This function also takes a `context`, which is used to manage and control the lifecycle of the request.

```go
// RegisterUserServiceHandlerFromEndpoint is same as RegisterUserServiceHandler but
// automatically dials to "endpoint" and closes the connection when "ctx" gets done.
func RegisterUserServiceHandlerFromEndpoint(ctx context.Context, mux *runtime.ServeMux, endpoint string, opts []grpc.DialOption) (err error) {
	conn, err := grpc.NewClient(endpoint, opts...)
	if err != nil {
		return err
	}
	defer func() {
		if err != nil {
			if cerr := conn.Close(); cerr != nil {
				grpclog.Errorf("Failed to close conn to %s: %v", endpoint, cerr)
			}
			return
		}
		go func() {
			<-ctx.Done()
			if cerr := conn.Close(); cerr != nil {
				grpclog.Errorf("Failed to close conn to %s: %v", endpoint, cerr)
			}
		}()
	}()
	return RegisterUserServiceHandler(ctx, mux, conn)
}

// RegisterUserServiceHandler registers the http handlers for service UserService to "mux".
// The handlers forward requests to the grpc endpoint over "conn".
func RegisterUserServiceHandler(ctx context.Context, mux *runtime.ServeMux, conn *grpc.ClientConn) error {
	return RegisterUserServiceHandlerClient(ctx, mux, NewUserServiceClient(conn))
}
```

Let's dive into RegisterUserServiceHandlerClient.
In this function, we found that mux appends handlers for each service call using callback functions.
These callbacks primarily handle annotations, invoke the actual gRPC service calls (e.g., `request_UserService_SignUp_0`), process metadata, and finally format the response in a RESTful manner (e.g., `forward_UserService_SignUp_0`).

```go
// RegisterUserServiceHandlerClient registers the http handlers for service UserService
// to "mux". The handlers forward requests to the grpc endpoint over the given implementation of "UserServiceClient".
// Note: the gRPC framework executes interceptors within the gRPC handler. If the passed in "UserServiceClient"
// doesn't go through the normal gRPC flow (creating a gRPC client etc.) then it will be up to the passed in
// "UserServiceClient" to call the correct interceptors. This client ignores the HTTP middlewares.
func RegisterUserServiceHandlerClient(ctx context.Context, mux *runtime.ServeMux, client UserServiceClient) error {
	mux.Handle(http.MethodPost, pattern_UserService_SignUp_0, func(w http.ResponseWriter, req *http.Request, pathParams map[string]string) {
		ctx, cancel := context.WithCancel(req.Context())
		defer cancel()
		inboundMarshaler, outboundMarshaler := runtime.MarshalerForRequest(mux, req)
		annotatedContext, err := runtime.AnnotateContext(ctx, mux, req, "/users.UserService/SignUp", runtime.WithHTTPPathPattern("/v1/users/signup"))
		if err != nil {
			runtime.HTTPError(ctx, mux, outboundMarshaler, w, req, err)
			return
		}
		resp, md, err := request_UserService_SignUp_0(annotatedContext, inboundMarshaler, client, req, pathParams)
		annotatedContext = runtime.NewServerMetadataContext(annotatedContext, md)
		if err != nil {
			runtime.HTTPError(annotatedContext, mux, outboundMarshaler, w, req, err)
			return
		}
		forward_UserService_SignUp_0(annotatedContext, mux, outboundMarshaler, w, req, resp, mux.GetForwardResponseOptions()...)
	})
	// ...
```

:::note
The difference between `RegisterUserServiceHandlerServer` and `RegisterUserServiceHandlerFromEndpoint` is documented in the generated file :

```go
// RegisterUserServiceHandlerServer registers the http handlers for service UserService to "mux".
// UnaryRPC     :call UserServiceServer directly.
// StreamingRPC :currently unsupported pending https://github.com/grpc/grpc-go/issues/906.
// Note that using this registration option will cause many gRPC library features to stop working. Consider using RegisterUserServiceHandlerFromEndpoint instead.
// GRPC interceptors will not work for this type of registration. To use interceptors, you must use the "runtime.WithMiddlewares" option in the "runtime.NewServeMux" call.
```

The main recommendation is to use `RegisterUserServiceHandlerFromEndpoint`, as using `RegisterUserServiceHandlerServer` may lead to compatibility issues or unexpected behavior with certain gRPC libraries.

:::

### `users/server/gateway_server.go`

In `NewGatewayServer`, a `runtime.NewServeMux` is created and assigned to the `http.Server.Handler`. Since this project uses a self-signed certificate authority (CA), the `http.Server.TLSConfig` must also be configured accordingly.

In `(g *GatewayServer) Start`, the service is registered using `RegisterUserServiceHandlerFromEndpoint` , with parameters including a lifecycle-managed context, the initialized `*runtime.ServeMux`, and the gRPC service's port and credentials.

Finally, because `cert.pem` and `key.pem` were already loaded during initialization, both parameters passed to `g.server.ListenAndServeTLS` can be empty strings.

```go
import "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"

func NewGatewayServer(tlsConfig *tls.Config, logger *zap.Logger) *GatewayServer {
	mux := runtime.NewServeMux(runtime.WithMiddlewares())

	return &GatewayServer{
		server: &http.Server{
			Handler:   mux,
			TLSConfig: tlsConfig,
		},
		logger: logger,
	}
}

func (g *GatewayServer) Start(ctx context.Context, httpAddr string, grpcAddr string) error {
	creds := credentials.NewTLS(g.server.TLSConfig)

	g.server.Addr = fmt.Sprintf(":%s", httpAddr)

	err := proto.RegisterUserServiceHandlerFromEndpoint(
		ctx,
		g.server.Handler.(*runtime.ServeMux),
		fmt.Sprintf(":%s", grpcAddr),
		[]grpc.DialOption{grpc.WithTransportCredentials(creds)},
	)
	if err != nil {
		return err
	}

	return g.server.ListenAndServeTLS("", "")
}
```

### `grpc_server.go`

The `logging.Option` passed to `NewGrpcServer` enables logging for key service events : (1) when a service is called and (2) when a service call is completed.

Therefore, after executing the `CreateUser` method in the program, the following logs will be generated :

```text
{"level":"info","timestamp":"2025-03-25T11:04:37.689+0800","caller":"logging/logging.go:220","msg":"started call","protocol":"grpc","grpc.component":"server","grpc.service":"users.UserService","grpc.method":"CreateUser","grpc.method_type":"unary","peer.address":"127.0.0.1:40030","grpc.start_time":"2025-03-25T11:04:37+08:00","grpc.time_ms":"0.011"}
{"level":"info","timestamp":"2025-03-25T11:04:37.772+0800","caller":"logging/logging.go:220","msg":"finished call","protocol":"grpc","grpc.component":"server","grpc.service":"users.UserService","grpc.method":"CreateUser","grpc.method_type":"unary","peer.address":"127.0.0.1:40030","grpc.start_time":"2025-03-25T11:04:37+08:00","grpc.code":"OK","grpc.time_ms":"83.598"}
```

`grpc.NewServer` in this project is a customized server built using [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware), with custom configurations for the logging, credentials, and timeout settings.

:::tip

The call to `reflection.Register(server)` enables gRPC server reflection, which allowing clients to dynamically discover available services and methods at runtime.

You can inspect the services exposed by the server using the following command :

```bash
grpcurl -plaintext localhost:8443 list
```

:::

```go
func NewGrpcServer(tlsConfig *tls.Config, logger *zap.Logger) *GrpcServer {
	opts := []logging.Option{
		logging.WithLogOnEvents(logging.StartCall, logging.FinishCall),
	}
	creds := credentials.NewTLS(tlsConfig)

	server := grpc.NewServer(
		grpc.ChainUnaryInterceptor(
			logging.UnaryServerInterceptor(InterceptorLogger(logger), opts...),
		),
		grpc.ChainStreamInterceptor(
			logging.StreamServerInterceptor(InterceptorLogger(logger), opts...),
		),
		grpc.Creds(creds),
		grpc.ConnectionTimeout(utils.TIMEOUT),
	)

	reflection.Register(server)
	return &GrpcServer{
		server: server,
		logger: logger,
	}
}

func (g *GrpcServer) Start(lis net.Listener) error {
	return g.server.Serve(lis)
}
```

### `server.go`

`NewServer` creates a server for the entire application. It first initializes the gRPC server and the HTTP proxy, then registers the controller implementation and binds it to the gRPC server.

The method `(s *Server) Run()` begins by launching the gRPC server in a separate goroutine, followed by starting the HTTP proxy in another goroutine. When shutting down via `(s *Server) Shutdown()`, the method first stops the HTTP proxy and then gracefully shuts down the gRPC server.

```go
func NewServer() (*Server, error) {
	// ...
	grpcServer := NewGrpcServer(tlsConfig, logger)
	gateway := NewGatewayServer(tlsConfig, logger)
	// ...
	proto.RegisterUserServiceServer(grpcServer.server, controller)
	// ...
}

func (s *Server) Run() error {
	// ...
	grpcPort := viper.GetString("ports.grpc")
	grpcLis, err := net.Listen("tcp", fmt.Sprintf(":%s", grpcPort))
	if err != nil {
		return err
	}
	errGroup.Go(func() error {
		return s.grpcServer.Start(grpcLis)
	})

	httpPort := viper.GetString("ports.http")
	errGroup.Go(func() error {
		return s.gateway.Start(ctx, httpPort, grpcPort)
	})
	// ...
}

func (s *Server) Shutdown(ctx context.Context) {
	// ...
	if err := s.gateway.server.Shutdown(ctx); err != nil {
		s.logger.Error("Failed to shutdown HTTP server:", zap.Error(err))
	}
	s.grpcServer.server.GracefulStop()
	// ...
}
```
