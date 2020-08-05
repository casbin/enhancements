# Distributed plugin

| Status |
| ------ |
| Draft  | 

## Overview

A plugin to ensure the consistency of multiple Casbin instances in distributed situations

## Motivation

Now the distribution of Casbin is implemented based on `Watcher`, but `Watcher` is to let all instances reload the database after the update occurs. This is a very resource-consuming operation, so `WatcherEx` (https://github.com/casbin/casbin/issues/398), you can notify other instances of the modified content, so as to achieve incremental updates and reduce the number of IOs. However, because `Watcher` notifies other instances after the update, it cannot be controlled for high concurrency mode, please refer to [https://github.com/casbin/casbin/issues/421](https://github.com/casbin/casbin/issues/421).


In addition, many people hope Casbin supports the master-slave mode[https://github.com/casbin/casbin/issues/402](https://github.com/casbin/casbin/issues/402). Their main purpose is to ensure data consistency in the case of multiple instances, so raft should be a good choice, this is a protocol that guarantees strong consistency. Through election of Leader, all other nodes are based on Leader to achieve strong consistency of the cluster. This algorithm has been adopted by many large projects such as etcd. Compared with master-slave mode, raft is a more efficient and reliable choice.


In order to facilitate Casbin access to the raft algorithm, we need to propose a new plugin. After discussion, we plan to name it **Dispatcher**. The main role of this plugin is to maintain the state of the model and persist the changed policy to the database.

## Design

```go
// Dispatcher is the interface for Casbin Dispatcher
struct Dispatcher interface{
    // AddPolicies adds policies rule to all instance.
    AddPolicies (sec string, ptype string, rules [][]string) error
    // RemovePolicies removes policies rule from all instance.
    RemovePolicies (sec string, ptype string, rules [][]string) error
    // RemoveFilteredPolicy removes policy rules that match the filter from all instance.
    RemoveFilteredPolicy(sec string, ptype string, fieldIndex int, fieldValues ...string) error
    // ClearPolicy clears all current policy in all instances
    ClearPolicy() error
    
    // SetEnforcer set up the instance that need to be maintained.
    // The parameter should be SyncedEnforced
    SetEnforcer(interface{})
}
```


Since different raft implementations provide many different functions, `Dispatcher` should only include the functions that Casbin-core needs to use, and leave the other parts to the plug-in to implement it by itself. This ensures that users can use the different features of different plug-ins as much as possible


The `Propose` is replaced by `AddPolicy`,`AddPolicies` etc. The `ConfChange` is not necessary for Casbin-core. It should be defined by the plug-in itself. In addition, there will be other roles in different raft implementations, such as `Learner` and `NotVoter`. We should not restrict this.


`Dispatcher` is to ensure the consistency of Casbin at runtime. It does not process the initial data at loading, still hands it to the adapter. Users need to ensure that the state of all instances is consistent before using `Dispatcher`. 


The following is an example of use
```go
e, _ := NewEnforcer("examples/basic_model.conf", "examples/basic_policy.csv")

// The initialization method of Dispatcher should be defined by the plugin itself
// the config may to include the address and ID information of other instances, 
// as well as the configuration information of raft,
d, err := NewDispatcher(config)
if err != nil {
    log.Fatal(err)
}
// In SetDispatcher(), d.SetEnforcer(e) will be called, no need to set it yourself
err := e.SetDispatcher(d)
if err != nil {
    log.Fatal(err)
}

// Then you can use Casbin as before，The newly added policy
// will be automatically synchronized to other instances
e.AddPolicy("alice", "data2", "read")
```
The `Dispatcher` is designed to replace the `Watcher` and provide a more secure and reliable distributed system to facilitate the integration of Casbin in large systems.