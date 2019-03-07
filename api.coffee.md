    superagent_ovh = require 'superagent-ovh'
    superagent = require 'superagent'
    superagent_prefix = require 'superagent-prefix'
    {debug} = (require 'tangible') 'sitadel-ovh:api'

Some OVH API returns the message inside the JSON response as a `message` field.
(Particularly the Portability API.)

    class OVHAPIError extends Error
      constructor: ({message,response},path,content) ->
        super [
          path
          JSON.stringify content
          response?.body?.message? ? message
        ].join ' → '

    catcher = (path,content) -> (error) ->
      error = new OVHAPIError error, path, content
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
          .catch catcher path, content
        body

      put: (path,content = {}) ->
        {body} = await @client
          .put path
          .send content
          .use @sign
          .catch catcher path, content
        body

      get: (path) ->
        {body} = await @client
          .get path
          .use @sign
          .catch catcher path, content
        body

      get_stream: (path) ->
        @client
          .get path
          .use @sign

    module.exports = API
