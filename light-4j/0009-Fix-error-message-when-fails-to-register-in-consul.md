### Summary
Currently when a configuration issue or some other problem results in the service 
failing to register in consul, the log message outputted to the user is that the 
service failed to bind to whatever port. This causes confusion. 

### Motivation


### Guide-level explanation
In the bind method of the server module, all exceptions are handled as output the 
message "failed to bind to port xxxx.". However consul registration is also implemented
in this method. It means that although the port is bound to xxxx port successfully but 
failed registered to consul, the output information would still be "failed to bind to 
port xxxx.". So this confusion was caused because the exception of registration was not 
handled separately. 

The modification is to put the code of registration into the try catch block separately.
And handling it as output the following message: 
```
"The service bound on " + port + " failed to register in consul."
```

The codes after modification:
```
if (config.enableRegistry) {
    try {
        // assuming that registry is defined in service.json, otherwise won't start
        // server.
        registry = SingletonServiceFactory.getBean(Registry.class);
        if (registry == null)
            throw new RuntimeException("Could not find registry instance in service map");
        // in kubernetes pod, the hostIP is passed in as STATUS_HOST_IP environment
        // variable. If this is null
        // then get the current server IP as it is not running in Kubernetes.
        String ipAddress = System.getenv(STATUS_HOST_IP);
        logger.info("Registry IP from STATUS_HOST_IP is " + ipAddress);
        if (ipAddress == null) {
            InetAddress inetAddress = Util.getInetAddress();
            ipAddress = inetAddress.getHostAddress();
            logger.info("Could not find IP from STATUS_HOST_IP, use the InetAddress " + ipAddress);
        }
        Map parameters = new HashMap<>();
        if (config.getEnvironment() != null)
            parameters.put("environment", config.getEnvironment());
        serviceUrl = new URLImpl("light", ipAddress, port, config.getServiceId(), parameters);
        registry.register(serviceUrl);
        if (logger.isInfoEnabled())
            logger.info("register service: " + serviceUrl.toFullStr());

        // start heart beat if registry is enabled
        SwitcherUtil.setSwitcherValue(Constants.REGISTRY_HEARTBEAT_SWITCHER, true);
        if (logger.isInfoEnabled())
            logger.info("Registry heart beat switcher is on");
    } catch (Exception e) {
        System.out.println("The service bound on " + port + " failed to register in consul.");
        if (logger.isInfoEnabled())
            logger.info("The service bound on " + port + " failed to register in consul.");
        return false;
    }
}
```
And even though the user can get the correct exception information. There are still problems 
that need to be resolved. That is, when the registration fails, the server is still running. 
So should we stop the server at this moment and output the "server stops due to registration 
failure."?
### Reference-level explanation


### Drawbacks


### Rationale and Alternatives


### Unresolved questions