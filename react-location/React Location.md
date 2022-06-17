Rerender 
- It's not necessary to rerender whole page when location change
-  Why rerender three times when navigate
	- rerender 2nd and 3rd both call `this.cleanMatchCache()`
	- 2nd rerender cause by `Router.useLayoutEffect` invocation 
	- 3rd rerender cuase by `preNotify`