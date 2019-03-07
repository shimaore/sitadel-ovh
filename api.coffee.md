    superagent_ovh = require 'superagent-ovh'
    superagent = require 'superagent'
    superagent_prefix = require 'superagent-prefix'
    {debug} = (require 'tangible') 'sitadel-ovh:api'

Some OVH API returns the message inside the JSON response as a `message` field.
(Particularly the Portability API.)

    class OVHAPIError extends Error
      constructor: ({message,response},method,path,content) ->
        super [
          method
          path
          if content? then JSON.stringify content else ''
          response?.body?.message? ? message
        ].join ' → '

    catcher = (method,path,content) -> (error) ->
      error = new OVHAPIError error, method, path, content
      debug.error 'OVHAPIError', error
      Promise.reject error

    class API
      constructor: (base,options) ->
        @sign = superagent_ovh.sign options
        @client = superagent.agent()
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
