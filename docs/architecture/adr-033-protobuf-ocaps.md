# ADR 033: Protobuf Module Object Capabilities

## Changelog

- 2020-10-05: Initial Draft

## Status

Proposed


## Abstract

> "If you can't explain it simply, you don't understand it well enough." Provide a simplified and layman-accessible explanation of the ADR.
> A short (~200 word) description of the issue being addressed.


## Context

> This section describes the forces at play, including technological, political, social, and project local. These forces are probably in tension, and should be called out as such. The language in this section is value-neutral. It is simply describing facts. It should clearly explain the problem and motivation that the proposal aims to resolve.
> {context body}


## Decision

> This section describes our response to these forces. It is stated in full sentences, with active voice. "We will ..."
> {decision body}

### Module Keys

```go
func Invoker(ctx context.Context, signer ModuleID, method string, args, reply interface{}, opts ...grpc.CallOption) error

type ModuleKey interface {
  grpc.ClientConn
  ID() ModuleID
}

type RootModuleKey struct {
  moduleName string
  msgInvoker Invoker()
}

type DerivedModuleKey struct {
  moduleName string
  path []byte
  msgInvoker Invoker()
}

type ModuleID struct {
  ModuleName string
  Path []byte
}

func (key ModuleID) Address() []byte {
  return AddressHash(key.ModuleName, key.Path)
}
```

### Inter-Module Communication

```go
func (k keeper) DoSomething(ctx context.Context, req *MsgDoSomething) (*MsgDoSomethingResponse, error) {
  // make a query
  bankQueryClient := bank.NewQueryClient(sdk.ModuleQueryConn)
  res, err := bankQueryClient.Balance(ctx, &QueryBalanceRequest{
    Denom: "foo",
    Address: ModuleKeyToBech32Address(k.moduleKey),
  })
  
  // send a msg
  bankMsgClient := bank.NewMsgClient(k.moduleKey)
  res, err := bankMsgClient.Send(ctx, &MsgSend{
    FromAddress: ModuleKeyToBech32Address(k.moduleKey),
    ToAddress: ...,
    Amount: ...,
  })

  // send a msg from a derived module account
  derivedKey := k.moduleKey.DerivedKey([]byte("some-sub-pool"))
  res, err := bankMsgClient.Send(ctx, &MsgSend{
    FromAddress: ModuleKeyToBech32Address(derivedKey),
    ToAddress: ...,
    Amount: ...,
  })
}
```

### Hooks

```proto
service Hooks {
  rpc AfterValidatorCreated(AfterValidatorCreatedRequest) returns (AfterValidatorCreatedResponse);
}

message AfterValidatorCreatedRequest {
  string validator_address = 1;
}

message AfterValidatorCreatedResponse { }
```


```go
func (k stakingKeeper) CreateValidator(ctx context.Context, req *MsgCreateValidator) (*MsgCreateValidatorResponse, error) {
  ...

  for moduleId := range k.modulesWithHook {
    hookClient := NewHooksClient(moduleId)
    _, _ := hooksClient.AfterValidatorCreated(ctx, &AfterValidatorCreatedRequest {ValidatorAddress: valAddr})
  }
  ...
}
```

### Module Registration and Requirements

```go
type Configurator interface {
  ModuleKey() RootModuleKey

  MsgServer() grpc.Server
  QueryServer() grpc.Server
  HooksServer() grpc.Server

  RequireMsgServer(msgServerInterface interface{})
  RequireQueryServer(queryServerInterface interface{})
}

type Provisioner interface {
  GetAdminMsgClientConn(msgServerInterface interface{}) grpc.ClientConn
  GetPluginClientConn(pluginServerInterface interface{}) func(ModuleID) grpc.ClientConn
}

type Module interface {
  Configure(Configurator)
  Provision(Provisioner)
}

type ModuleManager interface {
  GrantAdminAccess(module ModuleID, msgServerInterface interface{})
  GrantPluginAccess(module ModuleID, pluginServerInterface interface{})
}
```


## Consequences

> This section describes the resulting context, after applying the decision. All consequences should be listed here, not just the "positive" ones. A particular decision may have positive, negative, and neutral consequences, but all of them affect the team and project in the future.


### Backwards Compatibility

> All ADRs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The ADR must explain how the author proposes to deal with these incompatibilities. ADR submissions without a sufficient backwards compatibility treatise may be rejected outright.


### Positive

{positive consequences}

### Negative

{negative consequences}

### Neutral

{neutral consequences}


## Test Cases [optional]

Test cases for an implementation are mandatory for ADRs that are affecting consensus changes. Other ADRs can choose to include links to test cases if applicable.


## References

- {reference link}