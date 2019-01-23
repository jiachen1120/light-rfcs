### Summary
Existing configuration modules can only load config separately through 
separate configuration files for each module. However, some developers 
want to be able to configure all modules with a single configuration 
file. For example, now if we want to configure the service module and 
the mask module, the server.yml and mask.yml need to be provided. But 
the feature discussed in this document is to put all configuration 
information into a file called `application` extend with yaml, json or yml 
along with the following format:
```
server:
    # some contents for server
mask:
    # some contents for mask
```

### Motivation


### Guide-level explanation
1. SingleConfig class

    A new class called 'SingleConfig' is created in config module to handle 
    the single configuration file `application` extended with json, yaml, or
    yml. The main functions for it is to check whether load config from 
    `application.yaml` is valid and then load object config or map config from 
    `application.yaml`.

    1.1 Enable the function
    
    A flag called `singleConfigEnabled` is used to control whether this 
    configure way is enabled. The default value for it is true and can 
    be set through the method called setSingleConfigEnabled().
    
    ```
    private static boolean singleConfigEnabled = true;
    
    public static void setSingleConfigEnabled(boolean enable) {
        singleConfigEnabled = enable;
    }
    ``` 
    
    1.2 Validation check of single configuration way
        
    A method called `isPresent()` is used to check whether single configuration
    is valid. The first condition is whether the flag `singleConfigEnabled` is 
    set to true. The second condition is that the `application` configuration file 
    exists and can be loaded as the required map or object.
    
    The `application` file is loaded initially and cached.
    
    ```
    private static final String SINGLE_CONFIG = "application";

    private static final Map<String, Object> singleConfigMap = Config.getInstance().getJsonMapConfig(SINGLE_CONFIG);
    
    public static boolean isPresent() {
        return singleConfigEnabled && singleConfigMap != null;
    }
    ```
   
    The default value for `singleConfigEnabled` is true, which means that if 
    the user provides a valid application configuration file, the single file 
    configuration mode must be used first.

    1.3 Load configuration from single config
    
    Two method provided to load config from application file based on modules'
    name. If the configuration file cannot be loaded from a single configuration 
    file, it will continue to try to load from a separate configuration 
    file in the module.
    
    When getting the object configuration, I tend to get config map first and 
    then convert it to an object type. This may be more reliable than using 
    reflection to get getter method from the class.
    
    ```
    public static Map<String, Object> getMapConfigFromSingleConfig(String configName) {
        Map<String, Object> mapConfig = (Map<String, Object>)singleConfigMap.get(configName);
        if (mapConfig == null) {
            logger.info("Cannot load config from application file extended with json, yaml or yml for " + configName + " module, " +
                    "try to load it from " + configName + " file extended with json, yaml or yml");
            return null;
        }
        return mapConfig;
    }


    public static Object getObjectConfigFromSingleConfig(String configName, Class clazz) {
        Map<String, Object> mapConfig = getMapConfigFromSingleConfig(configName);
        return Config.convertMapToObj(mapConfig, clazz);
    }
    ```
    
2. Use SingleConfig in Config
    
    2.1 Steps to load config
        
    Check cache first, if no such config try to load from application config file if 
    available. if config still be null, load from general config file just like
    what we do before.
    ```
    public Object getJsonObjectConfig(String configName, Class clazz) {
        checkCacheExpiration();
        Object config = configCache.get(configName);
        if (config == null) {
            synchronized (FileConfigImpl.class) {
                config = configCache.get(configName);
                if (config == null) {
                    if (SingleConfig.isPresent()) {
                        config = SingleConfig.getObjectConfigFromSingleConfig(configName, clazz);
                    }
                    if (config == null) {
                        config = loadObjectConfig(configName, clazz);
                    }
                    if (config != null) configCache.put(configName, config);
                }
            }
        }
        return config;
    }
    ``` 

### Reference-level explanation


### Drawbacks


### Rationale and Alternatives


### Unresolved questions