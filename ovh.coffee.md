OVH
===

    API = require './api'

    seconds = 1000
    scheduler_wait = 9*seconds
    default_periods = 12
    period_duration = 10*seconds

    class OVH

Static properties and methods.

      @DDI: 'ddi'
      @LINE: 'line'
      @NUMBER: 'number'
      @TRUNK: 'trunk'
      @SERVICE: 'service'
      @FAX: 'fax'
      @PORTABILITY: 'portability'
      @REDIRECT: 'redirect'
      @RSVA: 'rsva'

      @Type: (t) ->
        assert.strictEqual 'string', typeof t, "OVH.Type expected a string, got #{t}"
        ({type}) -> type is t

Constructor

      constructor: (@db,base,options) ->
        @api = new API base, options

        @__queue = new Set
        @__scheduler()

Index for `cancel_porta`, `execute_porta`, `reschedule_porta`

        heal 'OVH index for OVH orders', @db.createIndex
          index: fields: [ 'billingAccount', 'orderId' ]

        return

      schedule_check: (billingAccount) ->
        @__queue.add billingAccount

      __scheduler: ->
        if @__queue.size is 0
          setTimeout (=> @__scheduler()), scheduler_wait
          return
        for billingAccount from @__queue
          try
            await @force_check billingAccount
            @__queue.delete billingAccount
        return


Create billing account
----------------------

      create_billingAccount: (account,meta) ->
        debug 'create_billingAccount', account, meta

        order = @order await @api.post "/order/telephony/new"

        await order.pay_with_fidelityAccount()
        order

      update_account: (billingAccount,data) ->
        await @api.put "/telephony/#{ec billingAccount}", data
        await @force_get billingAccount

Create line
-----------

      create_line: (billingAccount,meta) ->
        debug 'create_line', billingAccount, meta

        switch meta.country
          when 'fr'
            meta.type ?= 'nogeographic' # or 'geographic'
            meta.zone ?= '' # ZNE, ignored for nogeographic
          when 'ch'
            meta.type ?= 'geographic'
            meta.zone ?= '004122519xxxx' # Geneva

        meta.quantity ?= 1

        {serviceId} = await @api.get "/telephony/#{ec billingAccount}/serviceInfos"

        {cartId} = await @api.post "/order/cart",
          description: "create-line #{billingAccount}"
          ovhSubsidiary: 'FR'

        await @api.post "/order/cart/#{ec cartId}/assign"

        {itemId} = await @api.post "/order/cart/#{ec cartId}/telephony",
          duration: "P1M"
          planCode: @line_plan_code meta.country
          pricingMode: 'default'
          quantity: meta.quantity

        ###
        configuration = await @api.get "/order/cart/#{ec cartId}/item/#{ec itemId}/requiredConfiguration"
        ###

configuration is an array of:
required: true/false
allowedValues: []
type: "String" etc.
fields: null
label: "paymentMeanType"

        params = # for a line
          country: meta.country
          numberType: meta.type
          groupConfigId: serviceId
          billingAccount: billingAccount
          geographicArea: meta.zone
          # paymentMeanType:
          # paymentMeanId:
          # specialPrefix:
          directoriesPublication: false
          # meContactId:
          # TECH_ACCOUNT
          # ADMIN_ACCOUNT

        for label,value of params
          {id} = await @api.post "/order/cart/#{cartId}/item/#{itemId}/configuration", {label,value}

        order = @order await @api.post "/order/cart/#{ec cartId}/checkout",
          # autoPayWithPreferredPaymentMethod: true
          waiveRetractationPeriod: true

        ###
        contact = await @contact meta
        order = @order await @api.post "/order/telephony/#{ec billingAccount}/line",
          displayUniversalDirectories: [false]
          extraSimultaneousLines: [0]
          offers: ['priceplan.landLine.40.unlimited']
          ownerContactIds: [contact]
          retractation: false
          shippingContactId: contact
          types: [meta.type]
          zones: [meta.zone]
          quantity: meta.quantity # of the same configuration

        ###

        await order.pay_with_bankAccount()
        order

Create fax
----------

      create_fax: (billingAccount,meta) ->
        debug 'create_fax', billingAccount, meta

        meta.type ?= 'nogeographic' # or 'geographic'
        meta.zone ?= '' # ZNE, ignored for nogeographic
        meta.quantity ?= 1

Aussi: on ne sait commander que du fax non-géo ("ligne" pure), parce qu'il faut que le "contact" (je présume le shippingContactId) soit dans la bonne commune.

        contact = await @contact meta
        order = @order await @api.post "/order/telephony/#{ec billingAccount}/line",
          displayUniversalDirectories: [false]
          extraSimultaneousLines: [0]
          offers: ['sipfax']
          ownerContactIds: [contact]
          retractation: false
          shippingContactId: contact
          types: [meta.type]
          zones: [meta.zone]
          quantity: meta.quantity # of the same configuration

        await order.pay_with_bankAccount()
        order

Remove service
--------------

      remove_service: (ovh_id) ->
        debug 'remove_service', ovh_id

        [billingAccount,type,service] = ovh_id.split ':'
        return unless service?

        await @api
          .delete "/telephony/#{ec billingAccount}/#{OVH.SERVICE}/#{service}"
          .send
            reason: 'other'
            details: "Provisioning requested removal"
        return

Create portability
------------------

Particulier:
- callNumber (numéro à porter) (au format OVH ou CCNQ (E.164-sans-plus-sign) ou national)
- country
- desireDate _or_ executeAsSoonAsPossible
- lineToRedirectAliasTo
- offer = individual → RIO
- socialReason = individual
- type = landline

- streetName
- streetNumber
- city
- zip
- rio

- name
- firstName

- streetNumberExtra
- streetType

- floor
- door
- stair
- building

Pro:
- offer = company
- socialReason = professional → SIRET
- siret
pareil, mais SIRET au lieu de RIO

Corporation:
pareil, mais ni SIRET ni RIO?


Example responses:

(400)
```
{
  "message": "RIO '…' is not valid"
}
```

(400)
```
{
  "message": "RIO '…' does not match number '…'"
}
```

      validate_porta: (billingAccount,meta) ->
        debug 'validate_porta', billingAccount, meta
        meta.streetNumber ?= 0

        @api.post "/order/telephony/#{billingAccount}/portability",
            Object.assign {
              displayUniversalDirectory: false
              type: 'landline' # or 'special'
            }, meta

      create_porta: (billingAccount,meta) ->
        debug 'create_porta', billingAccount, meta
        meta.streetNumber ?= 0

        order = @order await @api.post "/order/telephony/#{billingAccount}/portability",
          Object.assign {
            displayUniversalDirectory: false
            type: 'landline' # or 'special'
          }, meta

        await order.pay_with_fidelityAccount()
        order

      cancel_porta: (billingAccount,orderId) ->
        debug 'cancel_porta', billingAccount, orderId
        O = @db.findAsyncIterable
          selector: {billingAccount,orderId}

        for await {id,Missing} from O when not Missing
          await @api.post "/telephony/#{billingAccount}/portability/#{id}/cancel"
        return

      execute_porta: (billingAccount,orderId) ->
        debug 'execute_porta', billingAccount, orderId
        O = @db.findAsyncIterable
          selector: {billingAccount,orderId}

        for await {id} from O
          await @api.post "/telephony/#{billingAccount}/portability/#{id}/execute"
        return

      changedate_porta: (billingAccount,orderId,date) ->
        debug 'changedate_porta', billingAccount, orderId, date
        O = @db.findAsyncIterable
          selector: {billingAccount,orderId}

        for await {id} from O
          await @api.post "/telephony/#{billingAccount}/portability/#{id}/changedate", {date}
        return

      relaunch_porta: (billingAccount,orderId,parameters) ->
        debug 'relaunch_porta', billingAccount, orderId
        O = @db.findAsyncIterable
          selector: {billingAccount,orderId}

        parameters = Object.entries(parameters).map ([key,value]) -> {key,value}

        for await {id} from O
          await @api.post "/telephony/#{billingAccount}/portability/#{id}/relaunch", {parameters}
        return

Create non-geographic number
----------------------------

      create_nongeo: (billingAccount,meta) ->
        debug 'create_nongeo', billingAccount, meta

- Rattachement d'une ligne sur un groupe existant
- Mise en place d'un numéro supplémentaire sur une ligne existante (domain: 0033972xxxxxx)
- 1 numero supplémentaire - 1 mois (domain: 0033972xxxxxx)
- Exécution immédiate de la commande en renonçant au droit de rétractation

        order = @order await @api.post "/order/telephony/#{ec billingAccount}/numberNogeographic",
          Object.assign {
            displayUniversalDirectory: false
            offer: 'alias'
            retractation: false
          }, meta

        await order.pay_with_bankAccount()
        order

Create geographic number
----------------------------

      create_geo: (billingAccount,meta) ->
        debug 'create_geo', billingAccount, meta

- Rattachement d'une ligne sur un groupe existant
- Mise en place d'un numéro supplémentaire sur une ligne existante (domain: dépend de la ZNE)
- 1 numéro géographique supplémentaire - 1 mois (domain: dépend de la ZNE)
- Exécution immédiate de la commande en renonçant au droit de rétractation

        order = @order await @api.post "/order/telephony/#{ec billingAccount}/numberGeographic",
          Object.assign {
            displayUniversalDirectory: false
            offer: 'alias'
            retractation: false
          }, meta

        await order.pay_with_bankAccount()
        order

Update number of channels on a line
-----------------------------------

      update_channels: (ovh_line,quantity) ->
        debug 'update_channels', ovh_line.serviceName, quantity
        order = @order await @api.post "/order/telephony/lines/#{ec ovh_line.serviceName}/updateSimultaneousChannels",
          {quantity}
        await order.pay_with_bankAccount()
        order

Actions
-------

      set_password: (ovh_line,password) ->
        {BillingAccount,serviceName} = ovh_line
        debug 'set_password', BillingAccount, serviceName
        await @api.post "/telephony/#{ec BillingAccount}/line/#{ec serviceName}/changePassword",
          {password}
        return

      change_feature_type: (ovh_number,featureType) ->
        {BillingAccount,serviceName} = ovh_number
        debug 'change_feature_type', BillingAccount, serviceName, featureType
        {taskId} = await @api.post "/telephony/#{ec BillingAccount}/number/#{ec serviceName}/changeFeatureType",
          {featureType}
        console.log 'change_feature_type', taskId
        @task BillingAccount, serviceName, taskId

      change_option: (ovh_service,data) ->
        {BillingAccount,serviceName} = ovh_service
        # data = Object.assign {}, ovh_service.Options, data
        debug 'change_option', BillingAccount, serviceName, data
        await @api.put "/telephony/#{ec BillingAccount}/line/#{ec serviceName}/options", data
        return

      change_destination: (ovh_number,number) ->
        {BillingAccount,featureType,serviceName} = ovh_number
        debug 'change_destination', BillingAccount, featureType, serviceName, number
        await @api.put "/telephony/#{ec BillingAccount}/#{ec featureType}/#{ec serviceName}/changeDestination", number

      change_directory: (ovh_service,infos) ->
        {BillingAccount,serviceName} = ovh_service
        debug 'change_directory', BillingAccount, serviceName, infos
        await @api.put "/telephony/#{ec BillingAccount}/service/#{ec serviceName}/directory", infos

Order
-----

      order: ({orderId}) ->
        debug 'order', orderId
        status = =>
          await @api.get "/me/order/#{ec orderId}/status"

        delivery = (periods = default_periods) =>
          switch current_status = await status()
            when 'checking', 'delivering'
              if periods > 0
                debug "Waiting for order #{orderId} #{current_status}"
                await sleep period_duration
                delivery periods-1 # tail recursion, hopefully
              else
                Promise.reject new Error "Order #{orderId} still #{current_status}"
            when 'delivered'
              true
            else # unknown, …
              Promise.reject new Error "Order #{orderId} #{current_status}"

For a new billingAccount, we get two details:
- "Groupe"
- "Mise en service d'un compte de facturation"
and they contain the same `domain`.

For now I don't understand the differences, if any, between `domain`, `service`, and `billingAccount`,
so I just assume this is what we get.

For a new "line" order, we get three details:
- "Frais de mise en service"
- "Forfait inclus vers les fixes en France et 40 pays et à la seconde vers les mobiles en France"
- "Exécution immédiate de la commande en renonçant au droit de rétractation"
les deux premiers ont `domain` avec le numéro (`0033972…`), par contre le dernier a `domain = "*"`.

For a new "non-geo":
- "1 numero supplémentaire - 1 mois"
- "Mise en place d'un numéro supplémentaire sur une ligne existante"
- "Exécution immédiate de la commande en renonçant au droit de rétractation"

        domain = =>
          details = await @api.get "/me/order/#{ec orderId}/details"
          domains = []
          for detail in details
            {domain} = await @api.get "/me/order/#{ec orderId}/details/#{ec detail}"
            domains.push domain unless domain is '*'
          domains.sort()[0]

        pay_with_fidelityAccount = =>
          await @api.post "/me/order/#{ec orderId}/payWithRegisteredPaymentMean",
            paymentMean: 'fidelityAccount'
          return

        pay_with_bankAccount = =>
          bankAccount = await @bankAccount()
          await @api.post "/me/order/#{ec orderId}/payWithRegisteredPaymentMean",
            paymentMean: 'bankAccount'
            paymentMeanId: bankAccount
          return

        {status,delivery,domain,pay_with_fidelityAccount,pay_with_bankAccount,id:orderId}

Task
----

      task: (billingAccount,serviceName,taskId) ->
        debug 'task', billingAccount, serviceName, taskId
        status = =>
          (await @api.get "/telephony/#{ec billingAccount}/#{OVH.SERVICE}/#{ec serviceName}/task/#{ec taskId}")
            .status

        completion = (periods = default_periods) =>
          switch current_status = await status()
            when 'doing', 'todo'
              if periods > 0
                debug "Waiting for task #{billingAccount} #{serviceName} #{taskId} #{current_status}"
                await sleep period_duration
                completion periods-1 # tail recursion, hopefully
              else
                Promise.reject new Error "Task #{billingAccount} #{serviceName} #{taskId} still #{current_status}"
            when 'done'
              true
            else # error, pause
              Promise.reject new Error "Task #{billingAccount} #{serviceName} #{taskId} #{current_status}"

        {status,completion,billingAccount,serviceName,id:taskId}

Tools
-----

Retrieve a `/telephony/` document using our local cache first.

      get: (path...) ->
        debug 'get', path

        id = path.join ':'

Use our local cache if present

        doc = await @db
          .get id
          .catch -> null
        return doc if doc?

Query OVH directly otherwise

        await @force_get path...

### force_get

Retrieve a `/telephony/` document using the API directly. Cache the result.

      __get: (path...) ->
        debug '__get', path

        id = path.join ':'

        try
          doc = await @api.get "/telephony/#{path.map(ec).join '/'}"
        catch error
          if error.status is 404
            await @db.merge id, Missing:true
          return Promise.reject error

        ['BillingAccount','type','Service'].forEach (name,i) ->
          doc[name] = path[i] if path.length > i
        doc._id = id
        doc.Missing = false
        await @db.update doc
        return await @db.get id

### force_get

`force_get(billingAccount)` or `force_get(billingAccount,cl,sv)`

      force_get: (path...) ->
        debug 'force_get', path

        data = await @__get path...
        return unless data?

        [billingAccount,cl,sv] = path

        get = (p) => @api.get "/telephony/#{ec billingAccount}/#{ec cl}/#{ec sv}/#{p}"

        switch cl
          when OVH.FAX
            try
              data.Settings = await get 'settings'

          when OVH.LINE
            try
              data.Options = await get 'options'

            try
              data.Channels = await get 'simultaneousChannelsDetails'

          when OVH.PORTABILITY
            try
              data.Status = await get 'status'

            try
              data.Documents = await get 'document'

            try
              data.CanBeCancelled = await get 'canBeCancelled'

            try
              data.CanBeExecuted = await get 'canBeExecuted'

            try
              data.DateCanBeChanged = await get 'dateCanBeChanged'

            try
              data.Relaunch = await get 'relaunch'

          when OVH.SERVICE
            try
              data.Directory = await get 'directory'

        await @db.update data
        return await @db.get data._id

### force_check

Retrieve all data linked to a billingAccount

      force_check: (billingAccount,services = default_services) ->
        debug 'force_check', billingAccount, services

        counter = 0

        await @db.info().catch => @db.create()

        ra = await @__get billingAccount

        for cl in services
          await do (cl) =>
            r1 = await @api.get "/telephony/#{ec billingAccount}/#{ec cl}"
            return unless r1
            for sv in r1
              await @force_get billingAccount, cl, sv
              counter++
            return

        if counter is 0
          console.error "billingAccount has no features!", billingAccount
        else
          console.log "billingAccount has #{counter} features", billingAccount
        return

      module.exports = OVH

      default_services = [
        # 'conference'
        OVH.DDI
        # 'easyHunting'
        # 'easyPabx'
        OVH.FAX
        OVH.LINE
        # 'miniPabx'
        OVH.NUMBER
        # 'ovhPabx'
        OVH.PORTABILITY
        OVH.REDIRECT
        OVH.RSVA
        # 'scheduler'
        # 'screen'
        OVH.SERVICE
        # 'timeCondition'
        OVH.TRUNK
        # 'voicemail'
        # 'vxml'
      ]

    {debug,heal} = (require 'tangible') 'sitadel-ovh:ovh'
    ec = encodeURIComponent
    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout
    assert = require 'assert'
