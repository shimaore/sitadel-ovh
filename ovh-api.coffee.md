    superagent_ovh = require 'superagent-ovh'
    superagent = require 'superagent'
    superagent_prefix = require 'superagent-prefix'

Some OVH API returns the message inside the JSON response as a `message` field.
(Particularly the Portability API.)

    class OVHAPIError extends Error
      constructor: ({message,response}) ->
        super response.body?.message? ? message

    catcher = (error) -> Promise.reject new OVHAPIError error

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
          .catch catcher
        body

      put: (path,content = {}) ->
        {body} = await @client
          .put path
          .send content
          .use @sign
          .catch catcher
        body

      get: (path) ->
        {body} = await @client
          .get path
          .use @sign
          .catch catcher
        body

      get_stream: (path) ->
        @client
          .get path
          .use @sign

    module.exports = API
