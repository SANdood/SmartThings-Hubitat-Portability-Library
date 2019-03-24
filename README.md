# SmartThings/Hubitat Portability Library (SHPL)
Portability library for developing code running on both SmartThings and Hubitat

## IMPORTANT - READ FIRST!
In both the SmartThings and the Hubitat development environments, `state` variables cannot be referenced in the metadata{} sections of an Application or a Device Handler. For this reason, the library implements two different approaches to figurring out and providing the current Hub Platform that code is running on. This SHPL provides metadata{}-safe calls that can be used anywhere (at a high overhead), and calls that can only be used in the runtime (which are relatively efficient on both platforms).

When using this library, `getHubPlatform()` ***MUST*** be called from the `installed()` method before accessing the state variables. Then you can use `state.hubPlatform`, `state.isST` and `state.isHE` in your runtime code.

### metadata{}-safe calls
The following 3 calls are safe to use anywhere within a Device Handler or Application
  - these can be called (e.g., if (getPlatform() == 'SmartThings'), or referenced (i.e., if (platform == 'Hubitat') )
  - performance of the non-native platform is horrendous, so it is best to use these only in the metadata{} section of a Device Handler or Application

~~~~
private String  getPlatform() { (physicalgraph?.device?.HubAction ? 'SmartThings' : 'Hubitat') }	// if (platform == 'SmartThings') ...
private Boolean getIsST()     { (physicalgraph?.device?.HubAction ? true : false) }					// if (isST) ...
private Boolean getIsHE()     { (hubitat?.device?.HubAction ? true : false) }						// if (isHE) ...
~~~~

### runtime code-only calls
The following 3 calls are ONLY for use within the Device Handler or Application runtime
  - they will throw an error at compile time if used within metadata, usually complaining that "state" is not defined
  - getHubPlatform() ***MUST*** be called from the installed() method, then use "state.hubPlatform" elsewhere
  - "if (state.isST)" is more efficient than "if (isSTHub)"
~~~~
private String getHubPlatform() {
    if (state?.hubPlatform == null) {
        state.hubPlatform = getPlatform()						// if (hubPlatform == 'Hubitat') ... or if (state.hubPlatform == 'SmartThings')...
        state.isST = state.hubPlatform.startsWith('S')			// if (state.isST) ...
        state.isHE = state.hubPlatform.startsWith('H')			// if (state.isHE) ...
    }
    return state.hubPlatform
}
private Boolean getIsSTHub() { (state.isST) }					// if (isSTHub) ...
private Boolean getIsHEHub() { (state.isHE) }					// if (isHEHub) ...
~~~~

### Tips & Tricks
Follows some tips and examples of how to apply SHPL to build portable code

#### sendHubCommand(hubAction) example:

~~~~
def example() {
    def hubAction
    if (state.isST) {
        hubAction = physicalgraph.device.HubAction.newInstance(
            method: 'GET',
            path: '/cgi-bin/template.cgi',
            headers: [ HOST: "${settings.meteoIP}:${settings.meteoPort}", 'Authorization': state.userpass, 'Accept': 'application/json,text/json', 'Content-Type': 'application/json', 'Accept-Charset': 'utf-8,iso-8859-1' ],
            query: ['template': "{\"timestamp\":${now()},\"version\":[mbsystem-swversion:1.0]," + (yesterday ? yesterdayTemplate : state.meteoTemplate), 'contenttype': 'application/json' ],
            null,
            [callback: hubActionCallback]
        )
    } else {
        hubAction = hubitat.device.HubAction.newInstance(
            method: 'GET',
            path: '/cgi-bin/template.cgi',
            headers: [ HOST: "${settings.meteoIP}:${settings.meteoPort}", 'Authorization': state.userpass, 'Accept': 'text/json,application/json', 'Content-Type': 'text/json', 'Accept-Charset': 'iso-8859-1,utf-8' ],
            query: ['template': "{\"timestamp\":${now()},\"version\":[mbsystem-swversion:1.0]," + (yesterday ? yesterdayTemplate : state.meteoTemplate), 'contenttype': 'text/json' ],
            null,
            [callback: hubActionCallback]
        )
    }
    try {
        sendHubCommand(hubAction)
    } catch (Exception e) {
    	if (debug) log.error "getMeteoWeather() sendHubCommand Exception ${e} on ${hubAction}"
    }
}
def hubActioCallback( hubResponse ) {
	// Note - don't try to define the Class for hubResponse above, as in "def hubActionCallback( physicalgraph.device.hubResponse hubResponse") -- it won't work.
    // Both platforms define hubResponse the same, so you don't need to classify it, just use as defined: hubResponse.status, hubResponse.json, hubResponse.body, etc.
}
~~~~

#### asynchttp* & encodeBase64()

~~~~
def example() {
    def userpassascii = meteoUser + ':' + meteoPassword
	if (state.isST) {
    	state.userpass = "Basic " + userpassascii.encodeAsBase64().toString()
    } else {
    	state.userpass = "Basic " + userpassascii.bytes.encodeBase64().toString()
    }
	String excludes = 'sources,minutely,daily,flags'
    String units = getTemperatureScale() == 'F' ? 'us' : (speed_units=='speed_kph' ? 'ca' : 'uk2')
    def apiRequest = [
        uri : "https://api.darksky.net",
        path : "/forecast/${settings.darkSkyKey}/${location.latitude},${location.longitude}",
        query : [ exclude : excludes, units : units ],
        contentType : "application/json"
    ]
    if (state.isST) {
    	include 'asynchttp_v1'
    	asynchttp_v1.get( asyncCallback, apiRequest )
    } else {
    	asynchttpGet( asyncCallback, apiRequest )
    }
}
def asyncCallback(response, data) {
	// do stuff
}
~~~~
