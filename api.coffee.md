    superagent_ovh = require 'superagent-ovh'
    superagent_prefix = require 'superagent-prefix'
    {debug} = (require 'tangible') 'sitadel-ovh:api'

    Request = require 'superagent'
    {HttpsAgent} = Agent = require 'agentkeepalive'

    http_agent = new Agent
      maxSockets: 100         # per host
      maxFreeSockets: 10      # per host
      timeout: 60000          # active socket
      freeSocketTimeout: 30000 # free socket
    https_agent = new HttpsAgent
      maxSockets: 100         # per host
      maxFreeSockets: 10      # per host
      timeout: 60000          # active socket
      freeSocketTimeout: 30000 # free socket

    pick_agent = (url) ->
      switch
        when url.match /^https:/
          Request.agent https_agent
        when url.match /^http:/
          Request.agent http_agent
        else
          Request

Some OVH API returns the message inside the JSON response as a `message` field.
(Particularly the Portability API.)

    class OVHAPIError extends Error
      constructor: ({message,response},method,path,content) ->
        msg = response?.body?.message ? message
        super [
          method
          path
          if content? then JSON.stringify content else ''
          response?.status
          response?.text
          msg
        ].join ' → '
        @ovh_message = msg
        return

    catcher = (method,path,content) -> (error) ->
      error = new OVHAPIError error, method, path, content
      debug.error 'OVHAPIError', error
      Promise.reject error

    class API
      constructor: (base,options) ->
        @sign = superagent_ovh.sign options
        @client = pick_agent base
          .use superagent_prefix base
          .accept 'json'
        return

      post: (path,content = {}) ->
        {body} = await @client
          .post path
          .send content
          .use @sign
          .catch catcher 'POST', path, content
        body

      put: (path,content = {}) ->
        {body} = await @client
          .put path
          .send content
          .use @sign
          .catch catcher 'PUT', path, content
        body

      get: (path) ->
        {body} = await @client
          .get path
          .use @sign
          .catch catcher 'GET', path
        body

      get_stream: (path) ->
        @client
          .get path
          .use @sign

    module.exports = API
