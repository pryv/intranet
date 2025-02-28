```javascript
class Global { // see name: AppState? AppContext? Config
  config: Config;
  logger: Logger;
  isReady: boolean;
  
  
	async init(overrideConfig: any): Promise<void> {
    // read file
    // fetch serviceInfo
    // apply override
    // breaks if called twice
  }

	/**
	 * to be called by any file/class (other than the one that calls init())
	 * before using any other method
	*/
  async isReady(void): Promise<void> {
  
  }
  
	// maybe put them on config property
	get(param: string): any {
    
  }

	set(paramName: string, value: any) {
    
  } 

	getLogger(prefix: string): Logger {
    return new Logger(prefix);
  }

	/**
	 * Usually called outside of 
	**/
	async notify(error: Error): void {
    
  }
    
}
```



### Usage

At the entry point of the app

```javascript
await globalObject.init();
```

Files/components that use it

```javascript
	// always call before .get
	await globalObject.isReady();
	new DB(config.get())
```

