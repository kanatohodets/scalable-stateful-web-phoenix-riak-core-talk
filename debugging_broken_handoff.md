# No handoff
This is my personal log from a debugging session early in my experimenting with Riak Core and Elixir.
I found that there was no handoff occurring on ring resize. __TL;DR__: some process names are special.
## Reproduce
The trouble is that handoff is not completing on resize. The ring is stuck
'awaiting'.

1. remove ring state (`make clean`)
2. start two node cluster, `dev_a` and `dev_b`
3. `dev_a`: `RicorEx.Service.ping` -> not part of any other ring yet, all pings
go to `dev_a`.
4. `dev_a`: `:riak_core.staged_join('dev_b@192.168.1.13')`
5. `dev_a`: `RicorEx.Service.ping`: notice all pings are directed to `dev_b`,
because the new ring isn't finalized yet, so all requests go to the 'old' one
(that is, just `dev_b`).
6. `dev_a`: `:riak_core_console.print_staged([])`
7. `dev_a`: `:riak_core_console.commit_staged([])`
8. at this point the ring gets stuck: no handoffs are conducted from `b->a`.
9. you can see the handoffs that _should_ happen like so:

```
    :riak_core_console.ring_status([])
    # or:
    {:ok, ring} = riak_core_ring_manager.get_my_ring
    :riak_core_ring.pending_changes(ring)
    # or:
    :riak_core_status.transfers
    # or:
    :riak_core_console.member_status([])
```

10. any of those methods will indicate that there are a bunch of indices
waiting to hand off to `dev_a` from `dev_b`.

## What is going on

`:riak_core_pending_changes(ring)` will show the chunk of the ring state that
relates to transfers (`next` in the state record). This will be full of things
like this:


     {1370157784997721485815954530671515330927436759040, :"dev_b@192.168.1.13",
       :"dev_a@192.168.1.13", [], :awaiting},

The empty list (actually, `ordset`) in the middle is meant to hold names of
VNode modules. When there are vnode modules in that list, it signals that the
transfer is complete. See `riak_core_ring.erl` line 1016 (state changes from
`awaiting` to `complete` if `ordsets:is_member(Mod, Mods)`).

These modules are added to that `ordset` in either `resize_transfer_complete`
on 1040 or in `transfer_complete` on 1612 (both `riak_core_ring.erl`).

### What calls `transfer_complete` or `resize_transfer_complete`?
`deletion_complete` or `handoff_complete`, which are external API functions for
`riak_core_ring`. `resize_transfer_complete/4` is also an external API
function.

### What calls `riak_core_ring:resize_transfer_complete/4`?
`riak_core_vnode.erl`, mostly.

### `riak_core_vnode.erl` and `riak_core_vnode_manager.erl` handoff triggers
The handoff is not triggered because of the 'error' clause in
`riak_core_handoff_manager` `maybe_trigger_handoff` 
The handoff is not triggered because of the 'error' clause in
`riak_core_handoff_manager` `maybe_trigger_handoff`. This means the {Mod, Idx}
combo was not found in the `handoff` dict kept in the `state` record for the
manager. The dict looks like this:
```
   {dict,0,16,16,8,80,48,{[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[]},{{[],[],[],[],[],[],[],[],[],[],[],[],[],[],[],[]}}}
```
and it should contain something like this:
```
    {'Elixir.RicorEx.Vnode',456719261665907161938651510223838443642478919680}
```
### Why isn't it in the dict?

This is part of the state record for VNode managers. Various things cause the
vnode manager to call `update_handoff` which calculates the new state and
returns it. `update_handoff` calls `riak_core_ring:ring_ready` and `should_handoff` to figure out what kind of
handoff should take place, and to whom. In our case it is apparent that it is
hitting the `false` of the `case` statement, or else there would be some values
in the dict.

### `should_handoff`

The problem is that `riak_core_node_watcher:nodes(App)` is returning an empty
list. so handoff never starts. Everything else looks ok. `app_for_vnode_module`
seems to be fine. 

### `riak_core_node_watcher:nodes(App)`

`nodes` is a thin wrapper around `internal_get_nodes` which just reads an ETS
table. Using `observer` its apparent that `ricor_ng_blorg` has stuff in these
tables, and RicorEx does _not_. Is there something weird going on with
application vs service name?

Disabling logging in the `vnode_manager`, since it seems the problem is really
in the `node_watcher`, in particular when it is registering services.

It looks like Elixir.RicorEx.Service is being added to the various ETS
tables.... and then removed right after! It's in the deleted set!

### Why is it in the deleted set?

`Elixir.RicorEx.Service` is added _twice_ to `node_watcher:add_service`. The
first time works fine and has a normal pid. The second time the pid is
_undefined_, and so Erlang immediately delivers a 'DOWN' notice for that
process, which results in `Elixir.RicorEx.Service` being removed from the set
of services. `ricor_blorg_ng` is added twice with two different (real) pids.

### Why is it added twice? Why is the second pid undefined?

`add_service` is called in two places as a result of a `service_up` call to
`node_watcher`. `service_up` in `node_watcher` is first called by MY
application in order to ... register the service. HOWEVER, it is also called by
`riak_core_ring_handler` in `ensure_vnodes_started`...AND it does
`erlang:whereis(SupName)`. That SupName is 
```
    SupName = list_to_atom(atom_to_list(App) ++ "_sup"),
```
...

The problem here is 'process name as an API': my supervisor wasn't named
`$appname_sup`; it was named `$AppName.Sup`, so `ensure_vnodes_started` did
`erlang:whereis($appname_sup)` and returned undefined. Because it returned
undefined, the monitor that was placed on the result of this `whereis` sends
`DOWN` right away.

### SO. TO FIX IT.

1. Application name in `mix.exs`: `:ricor_ex`.  
2. Supervisor process GETS A NAME, `:ricor_ex_sup`. Must be `$appname_sup`!
```
    defmodule RicorEx.Sup do
      use Supervisor

      def start_link do
        # riak_core appends _sup to the application name.
        Supervisor.start_link(__MODULE__, [], [name: :ricor_ex_sup])
      end

      def init(_args) do
        # riak_core appends _master to 'RicorEx.Vnode'
        children = [
          worker(:riak_core_vnode_master, [RicorEx.Vnode], name: RicorEx.Vnode_master, id: RicorEx.Vnode_master_worker),
        ]
        supervise(children, strategy: :one_for_one, max_restarts: 5, max_seconds: 10)
      end
    end
```
3. Application should look like so:

```
    defmodule RicorEx.App do
      use Application
      require Logger

      def start(_type, _args) do
        case RicorEx.Sup.start_link() do
          {:ok, pid} ->
            :ok = :riak_core.register([{:vnode_module, RicorEx.Vnode}])
            :ok = :riak_core_node_watcher.service_up(RicorEx.Service, self())
            {:ok, pid}
          {:error, reason} ->
            Logger.error("omg error while starting app: #{inspect(reason)}")
            {:error, reason}
        end
      end
    end
```
4. then it should work!

### All the process name based APIs
1. `$application_name ++ _sup` 
2. `$vnode_module_name ++ _master`
(haven't bumped into this one yet)
3. `$service_name ++ _entropy_manager`
