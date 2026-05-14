<!--- Application.cfc --->
<cfcomponent>

  <cfset THIS.apiKey = "YOUR_API_KEY">
  <cfset THIS.base   = "https://api.ip-block.com/v1">

  <!--- Returns true if the API is reachable --->
  <cffunction name="apiReachable" returntype="boolean">
    <cftry>
      <cfhttp url="#THIS.base#/ping" method="GET"
              timeout="1" throwonerror="false">
      <cfset local.data = deserializeJSON(cfhttp.fileContent)>
      <cfreturn (local.data.status EQ "ok")>
      <cfcatch><cfreturn false></cfcatch>
    </cftry>
  </cffunction>

  <cffunction name="onRequestStart" returntype="boolean">
    <cfif NOT apiReachable()>
      <cfreturn true> <!--- fail open --->
    </cfif>

    <cfset local.payload = serializeJSON({
      api_key    = THIS.apiKey,
      ip         = cgi.REMOTE_ADDR,
      site_id    = "ABCDEFGHIJKL",
      user_agent = cgi.HTTP_USER_AGENT,  <!--- optional --->
      referrer   = cgi.HTTP_REFERER      <!--- optional --->
    })>
    <cfhttp url="#THIS.base#/check" method="POST"
            timeout="2" throwonerror="false">
      <cfhttpparam type="header" name="Content-Type"
                   value="application/json">
      <cfhttpparam type="body" value="#local.payload#">
    </cfhttp>
    <cfset local.result = deserializeJSON(cfhttp.fileContent)>
    <cfif local.result.action EQ "block">
      <cflocation url="https://www.ip-block.com/blocked.php" addtoken="false">
    </cfif>
    <cfreturn true>
  </cffunction>

</cfcomponent>