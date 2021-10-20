# Job Manager

## Summary

Worker service which runs Linux processes.
[![CI](https://github.com/alxmke/jobs/actions/workflows/ci.yml/badge.svg?branch=master)](https://github.com/alxmke/jobs/actions/workflows/ci.yml)

## Outline

### Library

The Worker library is a Go package which makes Linux OS calls that manage a given set of processes. A Worker makes calls which start and stop jobs, stream and process their output, and handle errors.

Workers will keep their process statuses in a map in memory, which updates as the status of any given process changes. Jobs are tracked and queried using this in-memory map, and if this map and its Worker service  terminates, then so too consequently will the tracking of and control over its jobs.

As a convenience, the Worker writes the process output to a log file on disk in the /tmp directory (by default, configurable). Should conditions requrie it, due to spacial and other scale considerations, log files can be handled with more complex policies. As desired/required, any of the following could be adopted: logs can be compressed and/or archived locally, or stored using a cloud solution like AWS S3, or safely pruned based on some given criteria (date, or available space, for instance), or any combination of these (provided as optionals).

Resource control will be managed using the [cgroups](https://pkg.go.dev/github.com/containerd/cgroups) Go package.

```golang
// Command is a job request with given command name and arguments.
type Command struct {
	Name      string               // name/path of command
	Args      []string             // command arguments
	Resources specs.LinuxResources // resource control for process
}

// Status of the process.
type Status struct {
	Pid      int  // process ID
	ExitCode int  // exit code of process, -1 if not exited or if terminated by signal
	Exited   bool // has the process exited
}

// Job is a Linux process managed by a Worker.
type Job struct {
	ID     string    // job ID
	Cmd    *exec.Cmd // command being executed
	Status *Status   // status of the process
}

// Worker has operations which manage Jobs.
type Worker interface {
	// Start runs a Job.
	// - command: command to be run
	// returns the job ID and any error encountered.
	Start(command Command) (jobID string, err error)

	// Stop stops a running Job.
	// - ID: Job ID
	// returns any error encountered.
	Stop(jobID string) (err error)

	// Query checks the status of a Job.
	// - ID: Job ID
	// Returns process status and any error encountered.
	Query(jobID string) (status Status, err error)

	// Stream streams the process's output.
	// - ctx: job context
	// - ID: Job ID
	// returns channel to receive process's stdout/stderr, and any error encountered.
	Stream(ctx context.Context, jobID string) (logchan chan string, err error)
}
```
### API

The API will be supported via gRPC, and will provide the start/stop/query/stream link between Worker and user making use of both the UnaryInterceptor and StreamInterceptor. As per [gRPC's documentation](https://grpc.io/docs/guides/auth/), authentication and authorization will be handled using gRPC.

```protobuf
syntax = "proto3";
option go_package = "github.com/alxmke/jobs/worker/internal/proto";

message StartRequest {
  string name = 1;
  repeated string args = 2;
  optional struct resources = 3;
}

message StartResponse {
  string jobID = 1;
}

message StopRequest {
  string jobID = 1;
}

message StopResponse {
}

message QueryRequest {
  string jobID = 1;
}

message QueryResponse {
  int32 pid = 1;
  int32 exitCode = 2;
  bool exited = 3;
}

message StreamRequest {
  string jobID = 1;
}

message StreamResponse {
  string output = 1;
}

service WorkerService {
  rpc Start(StartRequest) returns (StartResponse);
  rpc Stop(StopRequest) returns (StopResponse);
  rpc Query(QueryRequest) returns (QueryResponse);
  rpc Stream(StreamRequest) returns (stream StreamResponse);
}
```

### Client	

The Client is a CLI which allows a user to make job requests to a Worker, mediated by gRPC. The CLI can query jobs which have been requested to check their status and stream their output.

## Security

### Transport

Mutual Transport Layer Security (mTLS), version 1.3, will assure privacy, data integrity, and a secure communication between the client and server.

### Authentication

Athentication will also be provided by mTLS. The following assets will be generated (using openssl 1.1.1f-1ubuntu2.8) to support authorization:
* CA private key and self-signed certificate
* private key and certificate signing request (CSR)
* signed certificate, based on Server CA private key and Server CSR

Authentication consists of:
* checking the certificate signature and finding a CA certificate with a subject field that matches the issuer field of the target certificate
* once the proper authority certificate is found, the validator checks the signature on the target certificate using the public key in the CA certificate.
* if the signature check fails, the certificate is invalid and the connection will not be established. 

The client and server each execute the same process to validate each other.

### Authorization

Authorization behaves as follows:
* user roles will be added to the client certificate, and the gRPC server interceptors will inspect the users' roles to authorize the user
* the available roles are reader and writer; the writer will be authorized start, stop, query and stream, and the reader will be authorized to query and stream
* a map contains the name of each gRPC method and each role authorized to execute 
* X.509 v3 extensions will be used to add the user role to the certificate by adding the extension attribute roleOid 1.2.840.10070.8.1 = ASN1:UTF8String to the CSR
* when the user certificate is created, user roles must be provided as the CA signs the the CSR.
* information after UTF8String: is encoded inside of the x509 certificate under the given OID

#### Certificates
* X.509
* Signature Algorithm: SHA-256 with RSA Encryption
* Public Key Algorithm: RSA Encryption
* RSA Public-Key: (4096 bit)
* roleOid: 1.2.840.10070.8.1 = ASN1:UTF8String (client)

## Scope and Trade-offs
Currently the client connects to only one Worker node, without considering load balancing and/or multiple server availabilities.
This could be scaled effectively by using a worker cluster with jobs routed by a load balancer. With this approach, a mapping of jobs to workers must be maintained and exposed to the load balancer so that the client can refer and route to jobs they've already requested.
Client-side load balancing would ease stress on what would otherwise be a single server-side load balancer, and provide robustness where there is no longer a single point of failure. This removes the latency of having to connect through this one single point, but also requires exposing a list of servers for the client to connect to, rather than the single load balancer access point, which in turn removes the simplicity of a single server access point from the client perspective. Traffic will also have to be established with each working node, rather than mediating it via  the load balancer access point.