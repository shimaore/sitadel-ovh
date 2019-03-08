    describe 'The module', ->
      it 'should load', ->
        require '..'
    describe 'The API', ->
      it 'should create', ->
        API = require '../api'
        new API 'https://example.org', {}
