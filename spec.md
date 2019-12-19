# Common Machine Interface (CMI)

Authors:

* Prashanth <<prashanth@sap.com>> (@prashanth26)
* Hardik Dodiya <<hardik.dodiya@sap.com>> (@hardikdr)
* Gaurav Gupta <<gaurav.gupta07@sap.com>> (@ggaurav10)
* Amshuman K R <<amshuman.rao.karaya@sap.com>> (@amshuman-kr)

Credits:

* This document is inspired from the [CSI spec documentation](https://github.com/container-storage-interface/spec/blob/master/spec.md).
* It is also inspired from an historic internal documentation provided by Stoyan Rachev <<s.rachev@sap.com>> (@stoyanr).

## Table of contents

TODO

## Notational Conventions

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119) (Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997).

The key words "unspecified", "undefined", and "implementation-defined" are to be interpreted as described in the [rationale for the C99 standard](http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf#page=18).

An implementation is not compliant if it fails to satisfy one or more of the MUST, REQUIRED, or SHALL requirements for the protocols it implements.
An implementation is compliant if it satisfies all the MUST, REQUIRED, and SHALL requirements for the protocols it implements.

## Terminology

| Term            | Definition                                       |
|-----------------|--------------------------------------------------|
| CR              | Custom Resource (CR) is defined by a cluster admin using the Kubernetes Custom Resource Definition  primitive. |
| VM              | A Virtual Machine (VM) provisioned and managed by a provider. It could also refer to a physical machine in case of a bare metal provider.|
| gRPC            | [gRPC Remote Procedure Calls](https://grpc.io/) |
| MCM             | [Machine Controller Manager (MCM)](https://github.com/gardener/machine-controller-manager) is the controller used to manage VMs as a Custom Resource (CR) in Kubernetes. The MCM is made up of two containers - `CMI-Client` and `CMI-Plugin`. |
| Machine         | Machine refers to a VM that is provisioned/managed by MCM. It typically describes the metadata used to store/represent a Virtual Machine |
| Node            | Native kubernetes `Node` object. The objects you get to see when you do a "kubectl get nodes". Although nodes can be either physical/virtual machines, for the purposes of our discussions it refers to a VM. |
| CMI-client      | CMI-Client contains the *core generic logic used to manage machines*. It doesn't contain any cloud specific code to manage VMs. It delegates the job of creating/updating/deleting VMs to the `CMI-Plugin` using gRPC primitives. |
| CMI-Plugin      | CMI-Plugin (or) Plugin (or) CMI-Server is the plugin responsible for accepting gRPC calls from CMI-Client and invoking provider specific functionality to do the same. A simple example could be creation/deletion of VM on the provider. |

## Objective

To define an industry standard “Container Machine Interface” (CMI) that will enable VM (cloud) providers to develop a plugin (CMI-Server) once and manage VMs for any Kubernetes cluster.

### Goals in MVP

The Container Machine Interface (CMI) will

* Enable VM provider authors to write one CMI compliant Plugin that “just works” to manage machines for any Kubernetes cluster using [MCM's CR objects](https://github.com/gardener/machine-controller-manager/blob/master/pkg/apis/machine/v1alpha1/types.go).
* Define API (RPCs) that enable:
  * Dynamic provisioning and deprovisioning of a VM.
  * Provisioning of VM must be configurable.
  * Getting the status of one/more VMs.
* Define plugin protocol RECOMMENDATIONS.
  * Describe a process by which a Supervisor configures a Plugin.
  * Container deployment considerations (`CAP_SYS_ADMIN`, mount namespace, etc.).

### Non-Goals in MVP

The Container Machine Interface (CMI) explicitly will not define, provide, or dictate:

* Specific mechanisms by which a Plugin Supervisor manages the lifecycle of a Plugin, including:
  * How to maintain state (e.g. what is provisioned, ready, etc.).
  * How to deploy, install, upgrade, uninstall, monitor, or respawn (in case of unexpected termination) Plugins.
* Protocol-level authentication and authorization.
* Packaging of a Plugin.

## Solution Overview

This specification defines an interface along with the minimum operational and packaging recommendations for a VM provider to implement a CMI compatible plugin.
The interface declares the RPCs that a plugin MUST expose: this is the **primary focus** of the CMI specification.

### Architecture

The primary focus of this specification is on the **protocol** between a CMI-Client and a Plugin.
It SHOULD be possible to ship CMI-Client compatible Plugins for a variety of deployment architectures.
A CMI-Client SHOULD be equipped to handle both centralized and headless plugins, as well as split-component and unified plugins.
Several of these possibilities are illustrated in the following figures.

```
                             MCM Architecture
+-------------------------------------------+
|                                           |
|  +------------+           +------------+  |
|  |   CMI      |   gRPC    |   CMI      |  |
|  |   Client   +----------->   Plugin   |  |
|  +------------+           +------------+  |
|                                           |
+-------------------------------------------+

Figure 1: CMI client and CMI Plugin together make up
the machine part of MCM. Both of them are typically expected to run
as two containers in a same pod. However, both might
run in different pods as well.
```

### Machine Lifecycle

```
       +-+                      +-+
       |X|   +----------------->| |
       +++   |                  +++
        |    |                   ^
        |    |                   | Delete
        |    |                   | Machine
        |    |                   |
 Create |    | Delete       +----+-----+
Machine |    | Machine      |  STOPPED |
        |    |              +----^-----+
        |    |                   |
        |    |                   | Shutdown
    +---v----+---+               | Machine
    |   CREATED  |---------------+
    +------------+
```

Figure 2: The lifecycle of a dynamically provisioned machine, from
creation to destruction.

The above diagrams illustrate a general expectation with respect to how a MCM manages the lifecycle of a machine via the API presented in this specification.

:warning: Plugins SHOULD expose a minimum set of a REQUIRED RPCs (`CreateMachine`, `DeleteMachine`). Other RPCs (like `GetMachineStatus`, `ShutDownMachine`, `ListMachines` and `GetVolumeIDs`) CAN be optionally implemented. However, it is highly recommended to implement these OPTIONAL RPCs for an optimized working of CMI Client. For more details on the optimizations provided for each OPTIONAL RPC, please refer the sections for each of the RPC descriptions.
Unsupported RPCs SHOULD return an appropriate error code that indicates such (e.g. `CALL_NOT_IMPLEMENTED`).

## Container Machine Interface

This section describes the interface between `CMI-Client` and `CMI-Plugins`.

### RPC Interface

A `CMI-Client` interacts with an `CMI-Plugin` through RPCs.
Each VM provider MUST provide two sets of RPCs:

* **Identity Service**: These are the set of identity RPCs to be implemented by the CMI-PLugin.
* **Machine Service**: The CMI-Plugin implement this sets of machine RPCs.

```protobuf

service Identity {
    rpc GetPluginInfo(GetPluginInfoRequest)
        returns (GetPluginInfoResponse) {}

    rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
        returns (GetPluginCapabilitiesResponse) {}

    rpc Probe (ProbeRequest)
        returns (ProbeResponse) {}
}

service Machine {
    rpc CreateMachine (CreateMachineRequest)
        returns (CreateMachineResponse) {}

    rpc DeleteMachine (DeleteMachineRequest)
        returns (DeleteMachineResponse) {}

    rpc GetMachineStatus (GetMachineStatusRequest)
        returns (GetMachineStatusResponse) {}

    rpc ListMachines (ListMachinesRequest)
    returns (ListMachinesResponse) {}

    rpc ShutDownMachine (ShutDownMachineRequest)
    returns (ShutDownMachineResponse) {}

    rpc GetVolumeIDs(GetVolumeIDsRequest)
        returns (GetVolumeIDsResponse) {}
}

```

#### Concurrency

In general the CMI-Client is responsible for ensuring that there is no more than one call “in-flight” per machine at a given time.
However, in some circumstances, the CMI-Client MAY lose state (for example when the CMI-Client crashes and restarts), and MAY issue multiple calls simultaneously for the same machine.
The plugin SHOULD handle this as gracefully as possible.
The error code `ABORTED` MAY be returned by the plugin in this case (see the [Error Scheme](#error-scheme) section for details).

#### Field Requirements

The requirements documented herein apply equally and without exception, unless otherwise noted, for the fields of all protobuf message types defined by this specification.
Violation of these requirements MAY result in RPC message data that is not compatible with all CMI-Clients, CMI-Plugins and/or CMI middleware implementations.
Make sure to not pass any sensitive data through plain text messages. Also make sure to not log any such sensitive data.

##### Size Limits

CMI defines general size limits for fields of various types (see table below).
The general size limit for a particular field MAY be overridden by specifying a different size limit in said field's description.
Unless otherwise specified, fields SHALL NOT exceed the limits documented here.
These limits apply for messages generated by both the CMI-Client and CMI-plugins.

| Size       | Field Type          |
|------------|---------------------|
| 128 bytes  | string              |
| 4 KiB      | map<string, string> |

##### `REQUIRED` vs. `OPTIONAL`

* A field noted as `REQUIRED` MUST be specified, subject to any per-RPC caveats; caveats SHOULD be rare.
* A `repeated` or `map` field listed as `REQUIRED` MUST contain at least 1 element.
* A field noted as `OPTIONAL` MAY be specified and the specification SHALL clearly define expected behavior for the default, zero-value of such fields.

Scalar fields, even REQUIRED ones, will be defaulted if not specified and any field set to the default value will not be serialized over the wire as per [proto3](https://developers.google.com/protocol-buffers/docs/proto3#default).

#### Retry Configuration

Any of the RPCs defined in this spec MAY timeout and MAY be retried.
The CMI-Client may choose the maximum timeout it is willing to wait for a call, how long it waits between retries, and how many time it retries (these values are not negotiated between plugin and CMI-Client).
Ideally timeout, sleep-time, number of retries, or even retry strategy (exponential or not) should be configurable at the CMI-Client.

Idempotency requirements ensure that a retried call with the same fields continues where it left off when retried.

### Error Scheme

All CMI API calls defined in this spec MUST return a [standard gRPC status](https://github.com/grpc/grpc/blob/master/src/proto/grpc/status/status.proto).
Most gRPC libraries provide helper methods to set and read the status fields.

The status `code` MUST contain a [canonical error code](https://github.com/grpc/grpc-go/blob/master/codes/codes.go). CMI-Clients MUST handle all valid error codes. Each RPC defines a set of gRPC error codes that MUST be returned by the plugin when specified conditions are encountered. In addition to those, if the conditions defined below are encountered, the plugin MUST return the associated gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Missing required field | 3 INVALID_ARGUMENT | Indicates that a required field is missing from the request. More human-readable information MAY be provided in the `status.message` field. | Caller MUST fix the request by adding the missing required field before retrying. |
| Invalid or unsupported field in the request | 3 INVALID_ARGUMENT | Indicates that the one or more fields in this field is either not allowed by the Plugin or has an invalid value. More human-readable information MAY be provided in the gRPC `status.message` field. | Caller MUST either replace the field with a valid or supported field or remove the invalid or unsupported field before retrying. |
| Permission denied | 7 PERMISSION_DENIED | The Plugin is able to derive or otherwise infer an identity from the secrets present within an RPC, but that identity does not have permission to invoke the RPC. | System administrator SHOULD ensure that requisite permissions are granted, after which point the caller MAY retry the attempted RPC. |
| Operation pending for machine | 10 ABORTED | Indicates that there is already an operation pending for the specified machine. In general the CMI-Client is responsible for ensuring that there is no more than one call "in-flight" per machine at a given time. However, in some circumstances, the CMI-Client MAY lose state (for example when the CMI-Client crashes and restarts), and MAY issue multiple calls simultaneously for the same machine. The Plugin, SHOULD handle this as gracefully as possible, and MAY return this error code to reject secondary calls. | Caller (CMI-Client) SHOULD ensure that there are no other calls pending for the specified machine, and then retry with after configured back off configuration (timeout, strategy, # of retries). |
| Call not implemented | 12 UNIMPLEMENTED | The invoked RPC is not implemented by the Plugin or disabled in the Plugin's current mode of operation. | Caller MUST NOT retry. Caller MAY call `GetPluginCapabilities` to discover Plugin capabilities in future |
| Not authenticated | 16 UNAUTHENTICATED | The invoked RPC does not carry secrets that are valid for authentication. | Caller SHALL either fix the secrets passed to the RPC, or otherwise regalvanize said secrets such that they will pass authentication by the Plugin for the attempted RPC, after which point the caller MAY retry the attempted RPC. |

The status `message` MUST contain a human readable description of error, if the status `code` is not `OK`.
This string MAY be surfaced by CMI-Client to end users.

The status `details` MUST be empty. In the future, this spec MAY require `details` to return a machine-parsable protobuf message if the status `code` is not `OK` to enable CMI-Client's to implement smarter error handling and fault resolution.

### Secrets Requirements

Secrets MAY be required by plugin to complete a RPC request.
A secret is a string map where the key identifies the name of the secret (e.g. "username" or "password"), and the value contains the secret data (e.g. "bob" or "abc123"). These `key-value` pairs are used to authenticate the request at the provider.
Each key MUST consist of alphanumeric characters, '-', '_' or '.'.
Each value MUST contain a valid string.
An VM provider MAY choose to accept binary (non-string) data by using a binary-to-text encoding scheme, like base64.
An VM provider SHALL advertise the requirements for required secret keys and values in documentation.
CMI-Client SHALL permit passing through the required secrets.
This information is sensitive and MUST be treated as such (not logged, etc.) by the CMI-Client.

### MachineClass Resources

MCM introduces the CRD `MachineClass`. This is a blueprint for creating machines that join a certain cluster as nodes in a certain role. The plugin only works with `MachineClass` resources that have the structure described here.

#### ProviderSpec

The `MachineClass` resource contains a `providerSpec` field that is passed in the `ProviderSpec` request field to CMI methods such as [CreateMachine](#createmachine). The `ProviderSpec` can be thought of as a machine template from which the VM specification must be adopted. It can contain key-value pairs of these specs. An example for these key-value pairs are given below.

| Parameter | Mandatory | Type | Description |
|---|---|---|---|
| `vmPool` | Yes | `string` | VM pool name, e.g. `TEST-WOKER-POOL` |
| `size` | Yes | `string` | VM size, e.g. `xsmall`, `small`, etc. Each size maps to a number of CPUs and memory size. |
| `rootFsSize` | No | `int` | Root (`/`) filesystem size in GB |
| `tags` | Yes | `map` | Tags to be put on the created VM |

Most of the `ProviderSpec` fields are not mandatory. If not specified, the plugin passes an empty value in the respective `Create VM` parameter.

The `tags` can be used to map a VM to its corresponding machine object's Name

The `ProviderSpec` is validated by methods that receive it as a request field for presence of all mandatory parameters and tags, and for validity of all parameters.

#### Secrets

The `MachineClass` resource also contains a `secretRef` field that contains a reference to a secret. The keys of this secret are passed in the `Secrets` request field to CMI methods.

The secret can contain sensitive data such as
- `cloud-credentials` secret data used to authenticate at the provider
- `cloud-init` scripts used to initialize a new VM. The cloud-init script is expected to contain scripts to initialize the Kubelet and make it join the cluster.

#### Identifying Cluster Machines

To implement certain methods, the plugin should be able to identify all machines associated with a particular Kubernetes cluster. This can be achieved using one/more of the below mentioned ways:

* Names of VMs created by the plugin are prefixed by the cluster ID specified in the [ProviderSpec](#providerspec).
* VMs created by the plugin are tagged with the special tags like `kubernetes.io/cluster` (for the cluster ID) and `kubernetes.io/role` (for the role), specified in the [ProviderSpec](#providerspec).
* Mapping `Resource Groups` to individual cluster.

### Identity Service RPC

Identity service RPCs allow a CMI-Client to query a CMI-Plugin for capabilities, health, and other metadata.
The general flow of the success case MAY be as follows (protos illustrated in YAML for brevity):

1. CMI-Client queries metadata via Identity RPC.

```
   # CMI-Client --(GetPluginInfo)--> Plugin
   request:
   response:
      name: org.foo.whizbang.super-plugin
      version: blue-green
      manifest:
        baz: qaz
```

2. CMI-Client queries available capabilities of the plugin.

```
   # CMI-Client --(GetPluginCapabilities)--> Plugin
   request:
   response:
     capabilities:
       - rpc:
           type: CREATE_MACHINE
       - rpc:
           type: DELETE_MACHINE
```

3. CMI-Client queries the readiness of the plugin.

```
   # CMI-Client --(Probe)--> Plugin
   request:
   response: {}
```

#### `GetPluginInfo`

This REQUIRED RPC allows the CMI-Client to query information about the Plugin. It returns the `name`, `version` and any additional information about the plugin.

```protobuf
message GetPluginInfoRequest {
    // Intentionally empty.
}

message GetPluginInfoResponse {
    // The name MUST follow domain name notation format
    // (https://tools.ietf.org/html/rfc1035#section-2.3.1). It SHOULD
    // include the plugin's host company name and the plugin name,
    // to minimize the possibility of collisions. It MUST be 63
    // characters or less, beginning and ending with an alphanumeric
    // character ([a-z0-9A-Z]) with dashes (-), dots (.), and
    // alphanumerics between.
    // This field is REQUIRED.
    string name = 1;

    // The version specifies the plugin version
    // Value of this field is opaque to the CMI-Client.
    // This field is REQUIRED.
    string version = 2;

    // manifest contains a map of key-value pairs to pass
    // any additonal information about the plugin.
    // Values are opaque to the CMI-Client.
    // E.g. key = start-time, value = timestamp (xx.xx.xxxx xx:xx:xx IST).
    // This field is OPTIONAL.
    map<string, string> manifest = 3;
}
```

##### GetPluginInfo Errors

If the plugin is unable to complete the GetPluginInfo call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `GetPluginCapabilities`

This REQUIRED RPC allows the CMI-Client to query the supported capabilities of the Plugin "as a whole": it is the grand sum of all capabilities.
All CMI-Plugins of the same version (see `version` of `GetPluginInfoResponse`) of the Plugin SHALL return the same set of capabilities.

```protobuf

message GetPluginCapabilitiesRequest {
    // Intentionally empty.
}

message GetPluginCapabilitiesResponse {
    // All the capabilities that the machine service supports.
    // This field is OPTIONAL.
    repeated PluginCapability capabilities = 1;
}

// Specifies a capability of the CMI-Plugin.
message PluginCapability {
    message RPC {
        enum Type {
            // UNKNOWN is used to specific an capability beyond the set
            // provided below
            UNKNOWN = 0;

            // CREATE_MACHINE tells that the plugin implements the
            // CreateMachine() RPC.
            // Plugin is REQUIRED to provide this capability.
            CREATE_MACHINE = 1;

            // DELETE_MACHINE tells that the plugin implements the
            // DeleteMachine() RPC.
            // Plugin is REQUIRED to provide this capability.
            DELETE_MACHINE = 2;

            // GET_MACHINE_STATUS tells that the plugin implements the
            // GetMachineStatus() RPC
            GET_MACHINE_STATUS = 3;

            // SHUTDOWN_MACHINE tells that the plugin implements the
            // ShutDownMachine() RPC
            SHUTDOWN_MACHINE = 4;

            // LIST_MACHINES tells that the plugin implements the
            // ListMachines() RPC
            LIST_MACHINE = 5;

            // GET_VOLUME_IDS tells if the plugin
            // implements the GetVolumeIDs() RPC
            GET_VOLUME_IDS = 6;
        }
        Type type = 1;
    }
    oneof type {
        // RPC that the plugin supports.
        RPC rpc = 1;
    }
}
```

##### GetPluginCapabilities Errors

If the plugin is unable to complete the GetPluginCapabilities call successfully, it MUST return a non-ok gRPC code in the gRPC status.

#### `Probe`

A Plugin is REQUIRED implement this RPC call.
The primary utility of the Probe RPC is to verify that the plugin is in a healthy and ready state.
If an unhealthy state is reported, via a non-success response, a CMI-Client MAY take action with the intent to bring the plugin to a healthy state.
Such actions MAY include, but SHALL NOT be limited to, the following:

* Restarting the plugin container, or
* Notifying the plugin supervisor.

The Plugin MAY verify that it has the right configurations, devices, dependencies and plugins in order to run and return a success if the validation succeeds.
The CMI-Client MAY invoke this RPC at any time.
A CMI-Client MAY invoke this call multiple times with the understanding that a plugin's implementation MAY NOT be trivial and there MAY be overhead incurred by such repeated calls.
The VM provider SHALL document guidance and known limitations regarding a particular Plugin's implementation of this RPC.
For example, the VM provider MAY document the maximum frequency at which its Probe implementation SHOULD be called.

```protobuf
message ProbeRequest {
    // Intentionally empty.
}

message ProbeResponse {
    // Readiness allows a plugin to report its initialization status back
    // to the CMI-Client. Initialization for some plugins MAY be time consuming
    // and it is important for a CMI-Client to distinguish between the following
    // cases:
    //
    // 1) The plugin is in an unhealthy state and MAY need restarting. In
    //    this case a gRPC error code SHALL be returned.
    // 2) The plugin is still initializing, but is otherwise perfectly
    //    healthy. In this case a successful response SHALL be returned
    //    with a readiness value of `false`. Calls to the plugin's
    //    Machine services MAY fail due to an incomplete
    //    initialization state.
    // 3) The plugin has finished initializing and is ready to service
    //    calls to its Machine services. A successful
    //    response is returned with a readiness value of `true`.
    //
    // This field is OPTIONAL. If not present, the caller SHALL assume
    // that the plugin is in a ready state and is accepting calls to its
    // Machine services (according to the plugin's reported
    // capabilities).
    .google.protobuf.BoolValue ready = 1;
}

```

##### Probe Errors

If the plugin is unable to complete the Probe call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CMI-Client MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Plugin not healthy | 9 FAILED_PRECONDITION | Indicates that the plugin is not in a healthy/ready state. | Caller SHOULD assume the plugin is not healthy and that future RPCs MAY fail because of this condition. |
| Missing required dependency | 9 FAILED_PRECONDITION | Indicates that the plugin is missing one or more required dependency. | Caller MUST assume the plugin is not healthy. |


### Machine Service RPC

- The plugin MUST have a unique way to map a `machine object` to a `VM` which triggers the deletion for the corresponding VM backing the machine object.
- The plugin SHOULD have a unique way to map the `ProviderSpec` of a machine-class to a unique `Cluster`. This avoids deletion of other machines, not backed by the CMI-Client.

#### `CreateMachine`

A Plugin is REQUIRED to implement this RPC and set the `CREATE_MACHINE` capability.
This RPC will be called by the CMI-Client to provision a new VM on behalf of the requesting machine object.

- This call requests the plugin to create a VM backing the machine-object.
- If VM backing the `MachineName` already exists, and is compatible with the specified `ProviderSpec` in the `CreateMachineRequest`, the Plugin MUST reply `0 OK` with the corresponding `CreateMachineResponse`.
- The plugin can OPTIONALY make use of the secrets supplied in the `Secrets` map in the `CreateMachineRequest` to communicate with the provider.
- The plugin can OPTIONALLY make use of the `LastKnownState` field to decode the state of the VM operation based on the last known state of the VM. This can be useful to restart/continue an operations which are mean't to be atomic.
- The plugin MUST have a unique way to map a `machine object` to a `VM`. This could be implicitly provided by the provider by letting you set VM-names (or) could be explicitly specified by the plugin using appropriate tags to map the same.
- This operation MUST be idempotent.

- The `CreateMachineResponse` returned by this method is expected to return
    - `ProviderID` that uniquely identifys the VM at the provider. This is expected to match with the node.Spec.ProviderID on the node object.
    - `NodeName` that is the expected name of the machine when it joins the cluster. It must match with the node name.
    - `LastKnownState` is an OPTIONAL field that can store details of the last known state of the VM. It can be used by future operation calls to determine current infrastucture state. This state is saved on the machine object.

```protobuf
message CreateMachineRequest {
    // MachineName is the name of the machine object CRD for whom a VM is to be created.
    // This field is REQUIRED.
    string MachineName = 1;

    // ProviderSpec is the configuration needed to create a machine in bytes.
    // Plugin should parse this raw data into pre-defined spec in their respective projects.
    // This field is REQUIRED.
    bytes ProviderSpec = 2;

    // Secrets is the map containing necessary credentials for cloud-provider to create the machine.
    // This field is OPTIONAL.
    map<string, bytes> Secrets = 3 [(cmi_secret) = true];

    // LastKnownState is any additional information (if required) that can used by the plugin.
    // It can be used by the plugin to recover from the last execution/error state.
    // Plugin should parse this raw data into it's desired format.
    // This field is OPTIONAL.
    bytes LastKnownState = 4;
}

message CreateMachineResponse {
    // ProviderID is the unique identification of the VM at the cloud provider.
    // This could be the same/different from req.Name.
    // ProviderID typically matches with the node.Spec.ProviderID on the node object.
    // Eg: gce://project-name/region/vm-ID
    // This field is REQUIRED.
    string ProviderID = 1;

    // NodeName is the name of the node-object registered to kubernetes.
    // This field is REQUIRED.
    string NodeName = 2;

    // LastKnownState is any additional information (if required) that can used by the plugin.
    // It can be the final state of the executing call.
    // Could be used by the plugin to recover from the current execution/error state in any future calls.
    // Plugin should return a raw bytes string.
    // This field is OPTIONAL.
    bytes LastKnownState = 3;
}
```

##### CreateMachine Errors

If the plugin is unable to complete the CreateMachine call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CMI-Client MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| gRPC Code | Condition | Description | Recovery Behavior | Auto Retry Required |
|-----------|-----------|-------------|-------------------|------------|
| 0 OK | Successful | The call was successful in creating/adopting a VM that matches supplied creation request. The `CreateMachineResponse` is returned with desired values |  | N |
| 1 CANCELED | Cancelled | Call was cancelled. Perform any pending clean-up tasks and return the call |  | N |
| 2 UNKNOWN | Something went wrong | Not enough information on what went wrong | Retry operation after sometime | Y |
| 3 INVALID_ARGUMENT | Re-check supplied parameters | Re-check the supplied `MachineName` and `ProviderSpec`. Make sure all parameters are in permitted range of values. Exact issue to be given in `.message` | Update providerSpec to fix issues. | N |
| 4 DEADLINE_EXCEEDED | Timeout | The call processing exceeded supplied deadline | Retry operation after sometime | Y |
| 5 NOT_FOUND |  |  |  |  |
| 6 ALREADY_EXISTS | Already exists but desired parameters doesn't match | Parameters of the existing VM don't match the ProviderSpec | Create machine with a different name | N |
| 7 PERMISSION_DENIED | Insufficent permissions | The requestor doesn't have enough permissions to create an VM and it's required dependencies | Update requestor permissions to grant the same | N |
| 8 RESOURCE_EXHAUSTED | Resource limits have been reached | The requestor doesn't have enough resource limits to process this creation request | Enhance resource limits associated with the user/account to process this | N |
| 9 PRECONDITION_FAILED | VM is in inconsistent state | The VM is in a state that is invalid for this operation | Manual intervention might be needed to fix the state of the VM | N |
| 10 ABORTED | Operation is pending | Indicates that there is already an operation pending for the specified machine | Wait until previous pending operation is processed | Y |
| 11 OUT_OF_RANGE | Resources were out of range  | The requested number of CPUs, memory size, of FS size in ProviderSpec falls outside of the corresponding valid range | Update request paramaters to request valid resource requests | N |
| 12 UNIMPLEMENTED | Not implemented | Unimplemented indicates operation is not implemented or not supported/enabled in this service. | Retry with an alternate logic or implement this method at the plugin. Most methods by default are in this state | N |
| 13 INTERNAL | Major error | Means some invariants expected by underlying system has been broken. If you see one of these errors, something is very broken. | Needs manual intervension to fix this | N |
| 14 UNAVAILABLE | Not Available | Unavailable indicates the service is currently unavailable. | Retry operation after sometime | Y |
| 15 DATALOSS |  |  |  |  |
| 16 UNAUTHENTICATED | Missing provider credentials | Request does not have valid authentication credentials for the operation | Fix the provider credentials | N |

The status `message` MUST contain a human readable description of error, if the status `code` is not `OK`.
This string MAY be surfaced by CMI-Client to end users.

#### `DeleteMachine`

A Plugin is REQUIRED to implement this RPC and set the `DELETE_MACHINE` capability.
This RPC will be called by the CMI-Client to deprovision a VM backed by the requesting machine object.

- If a VM corresponding to the specified machine-object's name does not exist or the artifacts associated with the VM do not exist anymore (after deletion), the Plugin MUST reply `0 OK`.
- The plugin SHALL only act on machines belonging to the cluster-id/cluster-name obtained from the `ProviderSpec`.
- The plugin can OPTIONALY make use of the secrets supplied in the `Secrets` map in the `DeleteMachineRequest` to communicate with the provider.
- The plugin can OPTIONALY make use of the VM unique ID (returned by the plugin on machine creation) passed in the `ProviderID` map in the `DeleteMachineRequest`.
- The plugin can OPTIONALLY make use of the `LastKnownState` field to decode the state of the VM operation based on the last known state of the VM. This can be useful to restart/continue an operations which are mean't to be atomic.
- This operation MUST be idempotent.
- The plugin must have a unique way to map a `machine object` to a `VM` which triggers the deletion for the corresponding VM backing the machine object.

- The `DeleteMachineResponse` returned by this method is expected to return
    - `LastKnownState` is an OPTIONAL field that can store details of the last known state of the VM. It can be used by future operation calls to determine current infrastucture state. This state is saved on the machine object.

```protobuf
message DeleteMachineRequest {
    // MachineName is the name of the machine object for which a VM is to be deleted.
    // This field is REQUIRED.
    string MachineName = 1;

    // ProviderSpec is needed to filter VMs based a cluster before deleting.
    // Plugin should parse this raw data into pre-defined spec in their respective projects.
    // This field is REQUIRED.
    bytes ProviderSpec = 2;

    // Secrets is the map containing necessary credentials for cloud-provider to delete the machine.
    // This field is OPTIONAL.
    map<string, bytes> Secrets = 3 [(cmi_secret) = true];

    // ProviderID is the unique identification of the VM at the cloud provider.
    // This could be the same/different from req.Name.
    // ProviderID typically matches with the node.Spec.ProviderID on the node object.
    // Eg: gce://project-name/region/vm-ID
    // This field is OPTIONAL.
    string ProviderID = 4;

    // LastKnownState is any additional information (if required) that can used by the plugin.
    // It can be used by the plugin to recover from the last execution/error state.
    // Plugin should parse this raw data into it's desired format.
    // This field is OPTIONAL.
    bytes LastKnownState = 5;
}

message DeleteMachineResponse {
    // LastKnownState is any additional information (if required) that can used by the plugin.
    // It can be the final state of the executing call.
    // Could be used by the plugin to recover from the current execution/error state in any future calls.
    // Plugin should return a raw bytes string.
    // This field is OPTIONAL.
    bytes LastKnownState = 1;
}
```

##### DeleteMachine Errors

If the plugin is unable to complete the DeleteMachine call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.

| gRPC Code | Condition | Description | Recovery Behavior | Auto Retry Required |
|-----------|-----------|-------------|-------------------|------------|
| 0 OK | Successful | The call was successful in deleting a VM that matches supplied deletion request. |  | N |
| 1 CANCELED | Cancelled | Call was cancelled. Perform any pending clean-up tasks and return the call |  | N |
| 2 UNKNOWN | Something went wrong | Not enough information on what went wrong | Retry operation after sometime | Y |
| 3 INVALID_ARGUMENT | Re-check supplied parameters | Re-check the supplied `MachineName` and make sure that it is in the desired format and not a blank value. Exact issue to be given in `.message` | Update `MachineName` to fix issues. | N |
| 4 DEADLINE_EXCEEDED | Timeout | The call processing exceeded supplied deadline | Retry operation after sometime | Y |
| 5 NOT_FOUND |  |  |  |  |
| 6 ALREADY_EXISTS |  |  |  |  |
| 7 PERMISSION_DENIED | Insufficent permissions | The requestor doesn't have enough permissions to delete an VM and it's required dependencies | Update requestor permissions to grant the same | N |
| 8 RESOURCE_EXHAUSTED |  |  |  |  |
| 9 PRECONDITION_FAILED | VM is in inconsistent state | The VM is in a state that is invalid for this operation | Manual intervention might be needed to fix the state of the VM | N |
| 10 ABORTED | Operation is pending | Indicates that there is already an operation pending for the specified machine | Wait until previous pending operation is processed | Y |
| 11 OUT_OF_RANGE |  |  |  |  |  |
| 12 UNIMPLEMENTED | Not implemented | Unimplemented indicates operation is not implemented or not supported/enabled in this service. | Retry with an alternate logic or implement this method at the plugin. Most methods by default are in this state | N |
| 13 INTERNAL | Major error | Means some invariants expected by underlying system has been broken. If you see one of these errors, something is very broken. | Needs manual intervension to fix this | N |
| 14 UNAVAILABLE | Not Available | Unavailable indicates the service is currently unavailable. | Retry operation after sometime | Y |
| 15 DATALOSS |  |  |  |  |
| 16 UNAUTHENTICATED | Missing provider credentials | Request does not have valid authentication credentials for the operation | Fix the provider credentials | N |

The status `message` MUST contain a human readable description of error, if the status `code` is not `OK`.
This string MAY be surfaced by CMI-Client to end users.

#### `GetMachineStatus`

A Plugin MUST implement this RPC call if it has `GET_MACHINE_STATUS` capability.
This RPC will be called by the CMI-Client to get the status of a machine.
This optional RPC helps in optimizing the working of the plugin by avoiding unwanted calls to `CreateMachine()` and `DeleteMachine()`.

- If a VM corresponding to the specified machine object's `MachineName` exists on provider the `GetMachineStatusResponse` fields are to be filled similar to the `CreateMachineResponse`.
- The plugin SHALL only act on machines belonging to the cluster-id/cluster-name obtained from the `ProviderSpec`.
- The plugin can OPTIONALY make use of the secrets supplied in the `Secrets` map in the `GetMachineStatusRequest` to communicate with the provider.
- The plugin can OPTIONALY make use of the VM unique ID (returned by the plugin on machine creation) passed in the `ProviderID` map in the `GetMachineStatusRequest`.
- This operation MUST be idempotent.

```protobuf
message GetMachineStatusRequest {
    // MachineName is name of the machine object from which the VM is to be identified.
    // The plugin is responsible to issuing a get call for a VM backed by this machine name.
    // This field is REQUIRED.
    string MachineName = 1;

    // ProviderSpec is needed to filter VMs based a cluster before getting the status.
    // Plugin should parse this raw data into pre-defined spec in their respective projects.
    // This field is REQUIRED.
    bytes ProviderSpec = 2;

    // Secrets is the map containing necessary credentials for cloud-provider to list the machines.
    // This field is OPTIONAL.
    map<string, bytes> Secrets = 3 [(cmi_secret) = true];

    // ProviderID is the unique identification of the VM at the cloud provider.
    // This could be the same/different from req.Name.
    // ProviderID typically matches with the node.Spec.ProviderID on the node object.
    // Eg: gce://project-name/region/vm-ID
    // This field is OPTIONAL.
    string ProviderID = 4;
}

message GetMachineStatusResponse {
    // ProviderID is the unique identification of the VM at the cloud provider.
    // This could be the same/different from req.Name.
    // ProviderID typically matches with the node.Spec.ProviderID on the node object.
    // Eg: gce://project-name/region/vm-ID
    // This field is REQUIRED.
    string ProviderID = 1;

    // NodeName is the name of the node-object registered to kubernetes.
    // This field is REQUIRED.
    string NodeName = 2;
}
```

##### GetMachineStatus Errors

If the plugin is unable to complete the GetMachineStatus call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.

| gRPC Code | Condition | Description | Recovery Behavior | Auto Retry Required |
|-----------|-----------|-------------|-------------------|------------|
| 0 OK | Successful | The call was successful in getting machine details for given machine `MachineName` |  | N |
| 1 CANCELED | Cancelled | Call was cancelled. Perform any pending clean-up tasks and return the call |  | N |
| 2 UNKNOWN | Something went wrong | Not enough information on what went wrong | Retry operation after sometime | Y |
| 3 INVALID_ARGUMENT | Re-check supplied parameters | Re-check the supplied `MachineName` and make sure that it is in the desired format and not a blank value. Exact issue to be given in `.message` | Update `MachineName` to fix issues. | N |
| 4 DEADLINE_EXCEEDED | Timeout | The call processing exceeded supplied deadline | Retry operation after sometime | Y |
| 5 NOT_FOUND | Machine isn't found at provider | The machine could not be found at provider | Not required | N |
| 6 ALREADY_EXISTS |  |  |  |  |
| 7 PERMISSION_DENIED | Insufficent permissions | The requestor doesn't have enough permissions to get details for the VM and it's required dependencies | Update requestor permissions to grant the same | N |
| 8 RESOURCE_EXHAUSTED |  |  |  |  |
| 9 PRECONDITION_FAILED | VM is in inconsistent state | The VM is in a state that is invalid for this operation | Manual intervention might be needed to fix the state of the VM | N |
| 10 ABORTED |  |  |  |  |
| 11 OUT_OF_RANGE | Multiple VMs found | Multiple VMs found with matching machine object names | Orphan VM handler to cleanup orphan VMs / Manual intervention maybe required if orphan VM handler isn't enabled.  | Y |
| 12 UNIMPLEMENTED | Not implemented | Unimplemented indicates operation is not implemented or not supported/enabled in this service. | Retry with an alternate logic or implement this method at the plugin. Most methods by default are in this state | N |
| 13 INTERNAL | Major error | Means some invariants expected by underlying system has been broken. If you see one of these errors, something is very broken. | Needs manual intervension to fix this | N |
| 14 UNAVAILABLE | Not Available | Unavailable indicates the service is currently unavailable. | Retry operation after sometime | Y |
| 15 DATALOSS |  |  |  |  |
| 16 UNAUTHENTICATED | Missing provider credentials | Request does not have valid authentication credentials for the operation | Fix the provider credentials | N |

The status `message` MUST contain a human readable description of error, if the status `code` is not `OK`.
This string MAY be surfaced by CMI-Client to end users.

#### `ShutDownMachine`

A Plugin MUST implement this RPC call if it has `SHUTDOWN_MACHINE` capability.
This RPC will be called by the CMI-Client to shutdown a particular machine.
This optional RPC MIGHT try to shutdown machines before deleting them. It might also be used in future for features such as restarting a machine.

- If a VM corresponding to the `MachineName` is present and has accepted termination request (or) is already in terminated state, then the Plugin MUST reply `0 OK`.
- The plugin SHALL only act on machines belonging to the cluster-id/cluster-name obtained from the `ProviderSpec`.
- The plugin can OPTIONALY make use of the secrets supplied in the `Secrets` map in the `ShutDownMachineRequest` to communicate with the provider.
- The plugin can OPTIONALY make use of the VM unique ID (returned by the plugin on machine creation) passed in the `ProviderID` map in the `ShutDownMachineRequest`.
- The plugin can OPTIONALLY make use of the `LastKnownState` field to decode the state of the VM operation based on the last known state of the VM. This can be useful to restart/continue an operations which are mean't to be atomic.
- This operation MUST be idempotent.

```protobuf
message ShutDownMachineRequest {
    // MachineName is the machine object name for whose VM, the shutdown call is to be invoked.
    // The plugin is responsible to issuing a shutdown call for a VM backed by this machine name.
    // This field is REQUIRED.
    string MachineName = 1;

    // ProviderSpec is needed to filter VMs based a cluster before shutting down.
    // Plugin should parse this raw data into pre-defined spec in their respective projects.
    // This field is REQUIRED.
    bytes ProviderSpec = 2;

    // Secrets is the map containing necessary credentials for cloud-provider to shut down the machine.
    // This field is OPTIONAL.
    map<string, bytes> Secrets = 3 [(cmi_secret) = true];

    // ProviderID is the unique identification of the VM at the cloud provider.
    // This could be the same/different from req.Name.
    // ProviderID typically matches with the node.Spec.ProviderID on the node object.
    // Eg: gce://project-name/region/vm-ID
    // This field is OPTIONAL.
    string ProviderID = 4;

    // LastKnownState is any additional information (if required) that can used by the plugin.
    // It can be used by the plugin to recover from the last execution/error state.
    // Plugin should parse this raw data into it's desired format.
    // This field is OPTIONAL.
    bytes LastKnownState = 5;
}

message ShutDownMachineResponse {
    // LastKnownState is any additional information (if required) that can used by the plugin.
    // It can be the final state of the executing call.
    // Could be used by the plugin to recover from the current execution/error state in any future calls.
    // Plugin should return a raw bytes string.
    // This field is OPTIONAL.
    bytes LastKnownState = 1;
}
```

##### ShutDownMachine Errors

If the plugin is unable to complete the GetMachineStatus call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.

| gRPC Code | Condition | Description | Recovery Behavior | Auto Retry Required |
|-----------|-----------|-------------|-------------------|------------|
| 0 OK | Successful | The termination request for the VM was accepted (or) the VM is already in terminated state. |  | N |
| 1 CANCELED | Cancelled | Call was cancelled. Perform any pending clean-up tasks and return the call |  | N |
| 2 UNKNOWN | Something went wrong | Not enough information on what went wrong | Retry operation after sometime | Y |
| 3 INVALID_ARGUMENT | Re-check supplied parameters | Re-check the supplied `MachineName` and make sure that it is in the desired format and not a blank value. Exact issue to be given in `.message` | Update `MachineName` to fix issues. | N |
| 4 DEADLINE_EXCEEDED | Timeout | The call processing exceeded supplied deadline | Retry operation after sometime | Y |
| 5 NOT_FOUND |  |  |  |  |
| 6 ALREADY_EXISTS |  |  |  |  |
| 7 PERMISSION_DENIED | Insufficent permissions | The requestor doesn't have enough permissions to shut down a VM and it's required dependencies | Update requestor permissions to grant the same | N |
| 8 RESOURCE_EXHAUSTED |  |  |  |  |
| 9 PRECONDITION_FAILED | VM is in inconsistent state | The VM is in a state that is invalid for this operation | Manual intervention might be needed to fix the state of the VM | N |
| 10 ABORTED | Operation is pending | Indicates that there is already an operation pending for the specified machine | Wait until previous pending operation is processed | Y |
| 11 OUT_OF_RANGE |  |  |  |  |  |
| 12 UNIMPLEMENTED | Not implemented | Unimplemented indicates operation is not implemented or not supported/enabled in this service. | Retry with an alternate logic or implement this method at the plugin. Most methods by default are in this state | N |
| 13 INTERNAL | Major error | Means some invariants expected by underlying system has been broken. If you see one of these errors, something is very broken. | Needs manual intervension to fix this | N |
| 14 UNAVAILABLE | Not Available | Unavailable indicates the service is currently unavailable. | Retry operation after sometime | Y |
| 15 DATALOSS |  |  |  |  |
| 16 UNAUTHENTICATED | Missing provider credentials | Request does not have valid authentication credentials for the operation | Fix the provider credentials | N |

The status `message` MUST contain a human readable description of error, if the status `code` is not `OK`.
This string MAY be surfaced by CMI-Client to end users.

#### `ListMachines`

A Plugin MUST implement this RPC call if it has `LIST_MACHINES` capability.
The Plugin SHALL return the information about all the machines associated with the `ProviderSpec`.
Make sure to use appropriate filters to achieve the same to avoid data transfer overheads.
This optional RPC helps in cleaning up orphan VMs present in the cluster. If not implemented, any orphan VM that might have been created incorrectly by the CMI-Client/Plugin (due to bugs in code/infra) might require manual clean up.

- If the Plugin succeeded in returning a list of `MachineName` with their corresponding `ProviderID`, then return `0 OK`.
- The `ListMachineResponse` contains a map of `MachineList` whose
    - Key is expected to contain the `ProviderID` &
    - Value is expected to contain the `MachineName` corresponding to it's kubernetes machine CR object
- The plugin can OPTIONALY make use of the secrets supplied in the `Secrets` map in the `ListMachinesRequest` to communicate with the provider.

```protobuf
message ListMachinesRequest {
    // ProviderSpec is the configuration needed to list machines.
    // Plugin should parse this raw data into pre-defined spec in their respective projects.
    // This field is REQUIRED.
    bytes ProviderSpec = 1;

    // Secrets is the map containing necessary credentials for cloud-provider to list the machines.
    // This field is OPTIONAL.
    map<string, bytes> Secrets = 2 [(cmi_secret) = true];
}

message ListMachinesResponse {
    // MachineList is the map of list of machines. Format for the map should be map<ProviderID, MachineName>.
    // This field is REQUIRED.
    map<string, string> MachineList = 1;
}
```

##### ListMachines Errors

If the plugin is unable to complete the ListMachines call successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST return the specified gRPC error code.
The CMI-Client MUST implement the specified error recovery behavior when it encounters the gRPC error code.

| gRPC Code | Condition | Description | Recovery Behavior | Auto Retry Required |
|-----------|-----------|-------------|-------------------|------------|
| 0 OK | Successful | The call for listing all VMs associated with `ProviderSpec` was successful. |  | N |
| 1 CANCELED | Cancelled | Call was cancelled. Perform any pending clean-up tasks and return the call |  | N |
| 2 UNKNOWN | Something went wrong | Not enough information on what went wrong | Retry operation after sometime | Y |
| 3 INVALID_ARGUMENT | Re-check supplied parameters | Re-check the supplied `ProviderSpec` and make sure that all required fields are present in their desired value format. Exact issue to be given in `.message` | Update `ProviderSpec` to fix issues. | N |
| 4 DEADLINE_EXCEEDED | Timeout | The call processing exceeded supplied deadline | Retry operation after sometime | Y |
| 5 NOT_FOUND |  |  |  |  |
| 6 ALREADY_EXISTS |  |  |  |  |
| 7 PERMISSION_DENIED | Insufficent permissions | The requestor doesn't have enough permissions to list VMs and it's required dependencies | Update requestor permissions to grant the same | N |
| 8 RESOURCE_EXHAUSTED |  |  |  |  |
| 9 PRECONDITION_FAILED |  |  |  |  |
| 10 ABORTED |  |  |  |  |
| 11 OUT_OF_RANGE |  |  |  |  |  |
| 12 UNIMPLEMENTED | Not implemented | Unimplemented indicates operation is not implemented or not supported/enabled in this service. | Retry with an alternate logic or implement this method at the plugin. Most methods by default are in this state | N |
| 13 INTERNAL | Major error | Means some invariants expected by underlying system has been broken. If you see one of these errors, something is very broken. | Needs manual intervension to fix this | N |
| 14 UNAVAILABLE | Not Available | Unavailable indicates the service is currently unavailable. | Retry operation after sometime | Y |
| 15 DATALOSS |  |  |  |  |
| 16 UNAUTHENTICATED | Missing provider credentials | Request does not have valid authentication credentials for the operation | Fix the provider credentials | N |

The status `message` MUST contain a human readable description of error, if the status `code` is not `OK`.
This string MAY be surfaced by CMI-Client to end users.

#### `GetVolumeIDs`

A Plugin MUST implement this RPC call if it has `GET_VOLUME_IDS` capability.
This RPC will be called by the CMI-Client to get the `VolumeIDs` for the list of `PersistantVolumes (PVs)` supplied.
This optional RPC helps in serailzied eviction of pods with PVs while draining of machines. This implies applications backed by PVs would be evicted one by one, leading to shorter application downtimes.

- On succesful returnal of a list of `Volume-IDs` for all supplied `PVs`, the Plugin MUST reply `0 OK`.
- The `GetVolumeIDsResponse` is expected to return a repeated list of `strings` consisting of the `VolumeIDs` for `PVSpec` that could be extracted.
- If for any `PV` the Plugin wasn't able to identify the `Volume-ID`, the plugin MAY chose to ignore it and return the `Volume-IDs` for the rest of the `PVs` for whom the `Volume-ID` was found.
- Getting the `VolumeID` from the `PVSpec` depends on the Cloud-provider. You can extract this information by parsing the `PVSpec` based on the `ProviderType`
    - https://github.com/kubernetes/api/blob/release-1.15/core/v1/types.go#L297-L339
    - https://github.com/kubernetes/api/blob/release-1.15//core/v1/types.go#L175-L257
- This operation MUST be idempotent.

```protobuf
message GetVolumeIDsRequest{
    // PVSpecsList is a list of PV specs for whom volume-IDs are required
    // Plugin should parse this raw data into pre-defined list of PVSpecs
    // This field is REQUIRED.
    bytes PVSpecList = 1;
}

message GetVolumeIDsResponse{
    // VolumeIDs is a list of VolumeIDs.
    // This field is REQUIRED.
    repeated string VolumeIDs = 1;
}
```

##### GetVolumeIDs Errors

| gRPC Code | Condition | Description | Recovery Behavior | Auto Retry Required |
|-----------|-----------|-------------|-------------------|------------|
| 0 OK | Successful | The call getting list of `VolumeIDs` for the list of `PersistantVolumes` was successful. |  | N |
| 1 CANCELED | Cancelled | Call was cancelled. Perform any pending clean-up tasks and return the call |  | N |
| 2 UNKNOWN | Something went wrong | Not enough information on what went wrong | Retry operation after sometime | Y |
| 3 INVALID_ARGUMENT | Re-check supplied parameters | Re-check the supplied `PVSpecList` and make sure that it is in the desired format. Exact issue to be given in `.message` | Update `PVSpecList` to fix issues. | N |
| 4 DEADLINE_EXCEEDED | Timeout | The call processing exceeded supplied deadline | Retry operation after sometime | Y |
| 5 NOT_FOUND |  |  |  |  |
| 6 ALREADY_EXISTS |  |  |  |  |
| 7 PERMISSION_DENIED |  |  |  |  |
| 8 RESOURCE_EXHAUSTED |  |  |  |  |
| 9 PRECONDITION_FAILED |  |  |  |  |
| 10 ABORTED |  |  |  |  |
| 11 OUT_OF_RANGE |  |  |  |  |  |
| 12 UNIMPLEMENTED | Not implemented | Unimplemented indicates operation is not implemented or not supported/enabled in this service. | Retry with an alternate logic or implement this method at the plugin. Most methods by default are in this state | N |
| 13 INTERNAL | Major error | Means some invariants expected by underlying system has been broken. If you see one of these errors, something is very broken. | Needs manual intervension to fix this | N |
| 14 UNAVAILABLE | Not Available | Unavailable indicates the service is currently unavailable. | Retry operation after sometime | Y |
| 15 DATALOSS |  |  |  |  |
| 16 UNAUTHENTICATED |  |  |  |  |

The status `message` MUST contain a human readable description of error, if the status `code` is not `OK`.
This string MAY be surfaced by CMI-Client to end users.

## Protocol

### Connectivity

* A CMI-Client SHALL communicate with a Plugin using gRPC to access the `Identity`, and `Machine` services.
  * proto3 SHOULD be used with gRPC, as per the [official recommendations](http://www.grpc.io/docs/guides/#protocol-buffer-versions).
  * All Plugins SHALL implement the REQUIRED Identity & Machine service RPCs.
    Support for OPTIONAL RPCs is optional.
* The CMI-Client SHALL provide the listen-address for the Plugin by way of the `CMI_ENDPOINT` environment variable.
  Plugin components SHALL create, bind, and listen for RPCs on the specified listen address.
  * Only Network Sockets MAY be used as endpoints.
* All supported RPC services MUST be available at the listen address of the Plugin.

### Security

* The CMI-Client operator and Plugin Supervisor SHOULD take steps to ensure that any and all communication between the CMI-Client and Plugin Service are secured according to best practices.
* Communication between a CMI-Client and a Plugin SHALL be transported over Network Sockets.
  * gRPC is compatible with Network Sockets; it is the responsibility of the CMI-Client operator and Plugin Supervisor to properly secure access to the Domain Socket using OS filesystem ACLs and/or other OS-specific security context tooling.
  * VM Provider’s supplying stand-alone Plugin controller appliances, or other remote components that are incompatible with Network Sockets MUST provide a software component that proxies communication between a Network Socket and the remote component(s).
    Proxy components transporting communication over IP networks SHALL be responsible for securing communications over such networks.
* Both the CMI-Client and Plugin SHOULD avoid accidental leakage of sensitive information (such as redacting such information from log files).

### Debugging

* Debugging and tracing are supported by external, CMI-independent additions and extensions to gRPC APIs, such as [OpenTracing](https://github.com/grpc-ecosystem/grpc-opentracing).

## Configuration and Operation

### General Configuration

* The `CMI_ENDPOINT` environment variable SHALL be supplied to the Plugin by the Plugin Supervisor.
* An operator SHALL configure the CMI-Client to connect to the Plugin via the listen address identified by `CMI_ENDPOINT` variable.
* With exception to sensitive data, Plugin configuration SHOULD be specified by environment variables, whenever possible, instead of by command line flags or bind-mounted/injected files.


#### Plugin Bootstrap Example

* Supervisor -> Plugin: `CMI_ENDPOINT=tcp://127.0.0.1:8080`.
* Operator -> CMI-Client: use plugin at endpoint `tcp://127.0.0.1:8080`.
* CMI-Client: monitor `127.0.0.1:8080`.
* Plugin: read `CMI_ENDPOINT`, create Network socket at specified path, bind and listen.
* CMI-Client: observe that socket now exists, establish connection.
* CMI-Client: invoke `GetPluginCapabilities`.

#### Filesystem

* Plugins SHALL NOT specify requirements that include or otherwise reference directories and/or files on the root filesystem of the CMI Client.
* Plugins SHALL NOT create additional files or directories adjacent to the Network socket specified by `CMI_ENDPOINT`; violations of this requirement constitute "abuse".
  * The Plugin Supervisor is the ultimate authority of the directory in which the Network socket endpoint is created and MAY enforce policies to prevent and/or mitigate abuse of the directory by Plugins.

### Supervised Lifecycle Management

* For Plugins packaged in software form:
  * Plugin Packages SHOULD use a well-documented container image format (e.g., Docker, OCI).
  * The chosen package image format MAY expose configurable Plugin properties as environment variables, unless otherwise indicated in the section below.
    Variables so exposed SHOULD be assigned default values in the image manifest.
  * A Plugin Supervisor MAY programmatically evaluate or otherwise scan a Plugin Package’s image manifest in order to discover configurable environment variables.
  * A Plugin SHALL NOT assume that an operator or Plugin Supervisor will scan an image manifest for environment variables.

#### Environment Variables

* Variables defined by this specification SHALL be identifiable by their `CMI_` name prefix.
* Configuration properties not defined by the CMI specification SHALL NOT use the same `CMI_` name prefix; this prefix is reserved for common configuration properties defined by the CMI specification.
* The Plugin Supervisor SHOULD supply all RECOMMENDED CMI environment variables to a Plugin.
* The Plugin Supervisor SHALL supply all REQUIRED CMI environment variables to a Plugin.

##### `CMI_ENDPOINT`

Network endpoint at which a Plugin SHALL host CMI RPC services. The general format is:

    {scheme}://{authority}:{endpoint}

The following address types SHALL be supported by Plugins:

    - tcp://127.0.0.1:8080
    - tcp://10.0.0.1:6443

This variable is REQUIRED.

#### Operational Recommendations

The Plugin Supervisor expects that a Plugin SHALL act as a long-running service vs. an on-demand, CLI-driven process.

Supervised plugins MAY be isolated and/or resource-bounded.

##### Logging

* Plugins SHOULD generate log messages to ONLY standard output and/or standard error.
  * In this case the Plugin Supervisor SHALL assume responsibility for all log lifecycle management.
* Plugin implementations that deviate from the above recommendation SHALL clearly and unambiguously document the following:
  * Logging configuration flags and/or variables, including working sample configurations.
  * Default log destination(s) (where do the logs go if no configuration is specified?)
  * Log lifecycle management ownership and related guidance (size limits, rate limits, rolling, archiving, expunging, etc.) applicable to the logging mechanism embedded within the Plugin.
* Plugins SHOULD NOT write potentially sensitive data to logs (e.g. secrets).

##### Available Services

* Plugin Packages MAY support all or a subset of CMI services; service combinations MAY be configurable at runtime by the Plugin Supervisor.
  * This specification does not dictate the mechanism by which mode of operation MUST be discovered, and instead places that burden upon the VM Provider.
* Misconfigured plugin software SHOULD fail-fast with an OS-appropriate error code.

##### Linux Capabilities

* Plugins SHOULD clearly document any additionally required capabilities and/or security context.

##### Cgroup Isolation

* A Plugin MAY be constrained by cgroups.

##### Resource Requirements

* VM Providers SHOULD unambiguously document all of a Plugin’s resource requirements.

### Deploying
* **Recommended:** The CMI-Client and Plugin are typically expected to run as two containers inside a common `Pod`.
* However, for the security reasons they could execute on seperate Pods provided they have a secure way to exchange data between them.
