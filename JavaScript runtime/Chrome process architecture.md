There are two kinds of Chrome processes:
- Renderer process
- Not Renderer process, eg. browser process, plugin process, plugin process, GPU process ...

## Renderer process

### Process management

Per tab runs in one renderer process if device resource enough, otherwise same site tab share one renderer process.

Cross-site iframe runs in a separated renderer process.

That makes sure one site cannot access data from other sites without consent.

### Thread

![[Pasted image 20220609192221.png]]

## Not renderer process

Those processes are singleton.

Those processes are consolidated into one process if device resources are short.