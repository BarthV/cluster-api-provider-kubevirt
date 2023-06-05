_quick update on this issue_

I've been working for a couple days on it, and I'm now quite confident to says that current implementation is creating major problems on the road.
I'll try to explain as clearly as I can what is blocking us & what should be changed to implement `qemu-guest-agent` based checks in the VM's virt-launcher pod.

## What is going on (during bootstrap check) ?

1. Machine reconciliation is triggered in `kubevirt_machinecontroller.go`
2. Reconcilier fetches & carries a global context with multiple compiled informations (vm, vmi, kubevirtcluster, kubevirtmachines, ssh keys ... )
3. Reconcilier builds the infra cluster kubernetes client
   * Depending the kubevirt topology (local or external), machine controller reuses existing capk controller client (based on `controller-runtime`) or builds a new one from a kubeconfig (based on `controller-runtime` too)
4. Bootstrap check is done in `machine.go` implementation that receive global context & manipulates this controller-runtime client to configure & trigger a SSH check
5. Bootstrap result is returned to reconciler 

## What is the problem ?

If we want to check bootstrap status using an exec command, we'd like the kubernetes client to support such a feature.

**But it seems that controller-runtime client is not able to run exec command on pods at all.
So the only (good) way to be able to trigger a pod exec from the code would be to rebuild a new kube client based on `k8s.io/client-go` (which supports spawning pod exec commands)**

Although it's pretty easy to say, such a change requires extracting `controller-runtime` client's configuration in order to stay compatible with local / external setups. This configuration must therefore be extracted after building (or reusing) the infrastructure client (during step 3).

Also, extracting kube client configuration is not as simple as it looks. If we're reusing local controller client, it's not only "parsing a kubeconfig file".

## Why it has a big impact ?

So far, I've identified & tested several things to implement to support such a change :

* we need to make the infra cluster configuration available to the in the controller global context
* this configuration cannot be extracted from an existing `controller-runtime` client. So it has to be created during the client build stage.
* the machine bootstrap checker should be aware of this configuration and use it when qemu-guest-agent check is asked.