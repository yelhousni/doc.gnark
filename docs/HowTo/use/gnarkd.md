---
description: gnarkd, a gnark proving and verifying server
---

# `gnarkd`

For multiple reasons (resource allocation, security, architecture) it may be useful to isolate
`gnark` as a service. We provide a minimalist `gnarkd` daemon (Go package `github.com/consensys/gnark/gnarkd`),
which exposes synchronous and asynchronous gRPC APIs to create and verify proofs.

!!! warning

    `gnarkd` is experimental and might not stay in `gnark` main repository in the future.

!!! note

    `gnarkd` only support Groth16 proofs. [Support for PlonK](https://github.com/ConsenSys/gnark/issues/82)
    is planned for `v0.5.0` release.

## Starting `gnarkd`

To start `gnarkd` from the command line:

```bash
cd gnark/gnarkd
go build
./gnarkd
```

To start `gnarkd` using a Docker image:

```bash
cd gnark/gnarkd
docker build -f gnarkd/Dockerfile.example -t gnarkd .
docker run -it --rm  -p9002:9002 -p9001:9001 --mount type=bind,source="$(pwd)"/circuits,target=/root/circuits --mount type=bind,source="$(pwd)"/certs,target=/root/certs gnarkd:latest
```

When `gnarkd` starts, it loads the circuits defined in `circuits/` directory. Circuits must be stored
in a separate directory, under a curve subdirectory.

!!! example

    `circuits/bn254/cubic` will contain `cubic.pk`, `cubic.vk`, and `cubic.r1cs`.

    `circuitID` (as needed in the APIs) is then `bn254/cubic`

`gnarkd` listens on 2 distinct TCP connections for the following:

* gRPC calls
* Receiving large witnesses on asynchronous calls.

When receiving witnesses on asynchronous calls, the server expects `jobID`|`witness`.

* `jobID` is returned by `CreateProveJob` and is a
    ([standard RFC 4122 UUID](https://tools.ietf.org/html/rfc4122)) on 16 byte server implementation
    using [`github.com/google/uuid`](https://github.com/google/uuid).
* `gnarkd` knows which witness size to expect via `r1cs.GetNbPublicWires`, `r1cs.GetNbSecretWires`,
    and `r1cs.SizeFrElement`.

## APIs

The following is the `protobuf` service from
[`gnarkd.proto`](https://github.com/ConsenSys/gnark/blob/master/gnarkd/pb/gnarkd.proto):

```protobuf
/*
 Provides services to compute and verify Groth16 proofs
 */
service Groth16 {
    // Prove takes circuitID and witness as parameter
    // this is a synchronous call and bypasses the job queue
    // it is meant to be used for small circuits, for larger circuits (proving time) and witnesses,
    // use CreateProveJob instead
    rpc Prove(ProveRequest) returns (ProveResult);


    // Verify takes circuitID, proof and public witness as parameter
    // this is a synchronous call
    rpc Verify(VerifyRequest) returns (VerifyResult);


    // CreateProveJob enqueue a job into the job queue with WAITING_WITNESS status
    rpc CreateProveJob(CreateProveJobRequest) returns (CreateProveJobResponse);

    // CancelProveJob does what it says it does.
    rpc CancelProveJob(CancelProveJobRequest) returns (CancelProveJobResponse);

    // ListProveJob does what it says it does.
    rpc ListProveJob(ListProveJobRequest) returns (ListProveJobResponse);

    // SubscribeToProveJob enables a client to get job status changes from the server
    // at connection start, server sends current job status
    // when job is done (ok or errored), server closes connection
    rpc SubscribeToProveJob(SubscribeToProveJobRequest) returns (stream ProveJobResult);
}
```

## Generating SDKs

gRPC clients can be generated for multiple languages (for example Go, or Rust) see the `protoc`
document for more info. For Go:

```bash
protoc --experimental_allow_proto3_optional --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative  pb/gnarkd.proto
```

## Example client

!!!example "Example from [`example.go`](https://github.com/ConsenSys/gnark/blob/master/gnarkd/client/example.go)."

    ```go
    // Set up a connection to the server.
    conn, err := grpc.Dial(address, grpc.WithTransportCredentials(credentials.NewTLS(config)))
    if err != nil {
        log.Fatal(err)
    }
    c := pb.NewGroth16Client(conn)

    // serialize witness
    var buf bytes.Buffer
    var w cubic.Circuit
    w.X.Assign(3)
    w.Y.Assign(35)
    witness.WriteFullTo(&buf, ecc.BN254, &w)

    // synchronous call
    _, _ = c.Prove(ctx, &pb.ProveRequest{
        CircuitID: "bn254/cubic",
        Witness:   buf.Bytes(),
    })

    // async call
    r, _ := c.CreateProveJob(ctx, &pb.CreateProveJobRequest{CircuitID: "bn254/cubic"})
    stream, _ := c.SubscribeToProveJob(ctx, &pb.SubscribeToProveJobRequest{JobID: r.JobID})

    done := make(chan struct{})

    // get notified when job status changes
    go func() {
        for {
            resp, err := stream.Recv()
            if err == io.EOF {
                done <- struct{}{}
                return
            }
            log.Printf("new job status: %s", resp.Status.String())
        }
    }()

    // send the witness (async call)
    go func() {
        conn, _ := tls.Dial("tcp", "127.0.0.1:9001", config)
        defer conn.Close()

        jobID, _ := uuid.Parse(r.JobID)
        bjobID, _ := jobID.MarshalBinary()
        conn.Write(bjobID)
        conn.Write(buf.Bytes())
    }()

    <-done
    ```
