Auto-call
---------

This module places calls (presumably towards a client or Centrex extension) and sends them into the socket we control.
It ensures data is retrieved and injected in the call.

This module also triggers calls from within a conference.

    pkg = require '../../package'
    @name = "#{pkg.name}:middleware:client:place-call"
    {debug,foot} = (require 'tangible') @name
    Moment = require 'moment-timezone'
    {UPDATE} = require 'red-rings/operations'

    Nimble = require 'nimble-direction'
    CouchDB = require 'most-couchdb'

    escape = (v) ->
      "#{v}".replace ',', ','

    make_params = (data) ->
      ("#{k}=#{escape v}" for own k,v of data).join ','

    now = (tz = 'UTC') ->
      Moment().tz(tz).format()

    @handler = handler = (cfg) ->

      if cfg.local_redis_client?
        elected = (key) ->
          name = "elected-#{key}"
          winner = await cfg.local_redis_client
            .setnx name, false
            .catch -> null
          if winner
            await cfg.local_redis_client
              .expire name, 60
              .catch -> yes
          else
            debug 'Lost the election.'
          return winner
      else
        elected = -> Promise.resolve true

* cfg.session.profile (string) Configuration profile that should be used to place calls towards client, for automated calls.

      profile = cfg.session?.profile
      unless profile?
        debug.dev 'Missing cfg.session.profile, not starting.'
        return

      sofia_profile = "huge-play-#{profile}-egress"
      context = "#{profile}-egress"

See huge-play/conf/freeswitch

      host = cfg.host
      p = cfg.profiles?[profile]
      if host and p?
        port = p.egress_sip_port ? p.sip_port+10000
        socket_port = p.socket_port ? 5721
      else
        debug 'Missing cfg.host or cfg.profiles[profile], not starting.', {profile}
        return

Note: if there are multiple profiles in use we will get in trouble at that point. (FIXME)

      local_server = "#{cfg.host}:#{p.ingress_sip_port ? p.sip_port}"
      client_server = "#{cfg.host}:#{p.egress_sip_port ? p.sip_port+10000}"
      debug 'Using', {local_server,client_server}

      {Reference} = cfg

Place Call
----------

The calls are automatically sent back to ourselves so that they can be processed like regular outbound calls.

This feature will call an extension (client-side number) and when the extension picks the call up, it will call the destination number (either an internal or external destination).

Event parameters:
- `_id`
- `endpoint`
- `caller` (with appropriate `endpoint_via` translations if necessary)
- `destination`
- `callee_name` (optional)
- `callee_num` (optional)
- `call_timeout` (optional)

      prov = new CouchDB (Nimble cfg).provisioning

      place_call = foot (_id,data) ->
        {endpoint,caller} = data

        data.callee_name ?= pkg.name
        data.callee_num ?= data.destination

FIXME The data sender must do resolution of the `endpoint_via` and associated translations????
ANSWER: Yes. And store the result in the field `caller`.

        debug 'Received place-call', data

A proper reference is required.

        return unless $ = _id?.match /^call:([\w-]+)$/
        my_reference = new Reference $[1]

Load additional data from the endpoint.

        endpoint_data = await prov.get("endpoint:#{endpoint}").catch -> null
        return unless endpoint_data?
        return if endpoint_data.disabled or endpoint_data.src_disabled

        {account,timezone} = endpoint_data

Ensure only one FreeSwitch server processes those.

        domain = endpoint_data.number_domain
        return unless domain?

Note that Centrex-redirect uses both the local-server and the client-server.

        is_remote = await cfg.is_remote domain, [local_server,client_server].join '/'
        return if is_remote

        debug 'place-call: Placing call'

Session Reference Data


        await my_reference.set_endpoint endpoint
        await my_reference.set_number_domain domain
        await my_reference.set_account account
        await my_reference.set_destination data.destination

        xref = "xref=#{my_reference.id}"
        params = make_params

These are used by `huge-play/middleware/client/setup`.

          session_reference: my_reference.id
          origination_context: context
          sip_invite_params: xref
          sip_invite_to_params: xref
          sip_invite_contact_params: xref
          sip_invite_from_params: xref

And `ccnq4-opensips` requires `X-En` for routing.

          'sip_h_X-En': data.endpoint

        sofia = "{#{params}}sofia/#{sofia_profile}/sip:#{caller}"

        command = "socket(127.0.0.1:#{socket_port} async full)"

        argv = [
          sofia
          "'&#{command}'"

dialplan

          'none'

context

          context

cid_name -- callee_name, shows on the caller's phone and in Channel-(Orig-)Callee-ID-Name

          data.callee_name

cid_num -- called_num, shows on the caller's phone and in Channel-(Orig-)Callee-ID-Number

          data.callee_num

timeout_sec

          data.call_timeout ? ''

        ].join ' '
        cmd = "originate #{argv}"

        return unless await elected my_reference.id

        debug "Calling #{cmd}"

        res = await cfg.api(cmd).catch (error) ->
          msg = error.stack ? error.toString()
          debug "originate: #{msg}"
          msg

The `originate` command will return when the call is answered by the callee (or an error occurred).

        debug "Originate returned", res
        if res[0] is '+'
          debug 'caller-connected', my_reference.id
        else
          debug.dev 'caller-failed', my_reference.id

        debug 'Session state:', data.tags

Call to conference
------------------

Parameters:
- `_id`
- `endpoint`
- `conference` (was `name`)
- `destination`

      prov = new CouchDB (Nimble cfg).provisioning

      call_to_conference = foot (_id,data) ->
        {endpoint,destination} = data
        debug 'Received call-to-conference', data, local_server

        name = data.conference ? data.name

A proper reference is required.

        return unless $ = _id?.match /^call:([\w-]+)$/
        my_reference = new Reference $[1]

Ensure we are co-located with the FreeSwitch instance serving this conference.

        is_remote = await cfg.is_remote name, local_server
        return if is_remote

Load additional data from the endpoint.

        endpoint_data = await prov.get("endpoint:#{endpoint}").catch -> null
        return unless endpoint_data?
        return if endpoint_data.disabled or endpoint_data.src_disabled

        {account,timezone} = endpoint_data

Try to get the asserted number, assuming Centrex.

        number_data = await prov.get("number:#{endpoint}").catch -> {}
        calling_number = number_data.asserted_number ? endpoint.split('@')[0]

        {language} = data
        language ?= number_data.language
        language ?= endpoint_data.language

        timezone ?= data.timezone if data.timezone?

Duplicated from exultant-song (FIXME)

        debug 'call-to-conference: Placing call'

Session Reference Data

        await my_reference.set_account account
        await my_reference.set_endpoint endpoint
        await my_reference.set_number_domain endpoint.number_domain
        await my_reference.set_call_options
          group_confirm_key: '5' # if `exec`, `file` holds the application and parameters; otherwise, one or more chars to confirm
          group_confirm_file: 'phrase:conference:confirm:5' # defaults to `silence`
          group_confirm_error_file: 'phrase:conference:confirm:5'
          group_confirm_read_timeout: 15000 # defaults to 5000
          group_confirm_cancel_timeout: false
          language: language

Call it out

        xref = "xref:#{my_reference.id}"
        params = make_params

          sip_invite_params: xref
          sip_invite_to_params: xref
          sip_invite_contact_params: xref
          sip_invite_from_params: xref

          origination_caller_id_number: calling_number

        sofia = "{#{params}}sofia/#{sofia_profile}/sip:#{destination}@#{host}:#{port}"
        cmd = "originate #{sofia} &conference(#{name}++flags{})"

        return unless await elected my_reference.id

        debug "Calling #{cmd}"
        res = await cfg.api(cmd).catch (error) ->
          msg = error.stack ? error.toString()
          debug "conference: #{msg}"
          msg

        debug "Conference returned", res

Queuer place call
-----------------

The `body` should contains:
- `_id` (unique id for the request)
- `agent` (string)
- `destination` (string)
- `tags` (array)

      create_queuer_call = foot (_id,body) ->
        {queuer} = cfg
        {Agent} = queuer

        unless queuer? and Agent?
          debug 'create-queuer-call: no queuer'
          return

        unless body? and body.agent? and body.destination?
          debug 'create-queuer-call: invalid content', body
          return

        agent = new Agent queuer, body.agent

        is_remote = await cfg.is_remote agent.domain, [local_server,client_server].join '/'
        if is_remote
          debug 'create-queuer-call: not handled on this server', body
          return

Note: `create_egress_call` does not use the `_id` provided (because the same call might be submitted to multiple agents, so it needs to generate a new one every time).
So we're electing based on the complete `call:…` reference, and there's no need to rewrite the `_id` field.

        return unless await elected _id

        await queuer.create_egress_call_for agent, body
        return

      {place_call,call_to_conference,create_queuer_call}

    @server_pre = ->

      H = handler @cfg
      return unless H?

      @cfg.rr
      .receive 'call:*', @most_shutdown
      .filter ({op,doc,deleted}) -> op is UPDATE and doc? and not deleted
      .forEach (msg) ->

        switch

`create_queuer_call`
- `_id` (unique id for the request: "call:<unique>")
- `agent` (string)
- `destination` (string)
- `tags` (array)

          when msg.doc.agent?
            H.create_queuer_call msg.id, msg.doc

`call_to_conference`
- `_id` ("call:<xref>")
- `endpoint`
- `conference` (was `name`)
- `destination`

          when msg.doc.conference?
            H.call_to_conference msg.id, msg.doc

`place_call`
- `_id` ("call:<xref>")
- `endpoint`
- 'caller' (with appropriate `endpoint_via` translations if necessary)
- `destination`
- `callee_name` (optional)
- `callee_num` (optional)
- `call_timeout` (optional)

          when msg.doc.endpoint?
            H.place_call msg.id, msg.doc

      return

Click-to-dial (`place-call`)
----------------------------

    @include = ->

Force the destination for `place-call` calls (`originate` sets `Channel-Destination-Number` to the value of `Channel-Caller-ID-Number`).

      destination = await @reference.get_destination()
      if destination?

        @destination = destination
        await @reference.set_destination null

Also, do not wait for an ACK, since we're calling out (to the "caller"),
and therefor the call is already connected by the time we get here.

        @session.sip_wait_for_aleg_ack = false

Finally, generate a P-Charge-Info header so that the SBCs will allow the call through.

        account = await @reference.get_account()
        if account?
          await @export 'sip_h_P-Charge-Info': "sip:#{account}@#{@cfg.host}"

        @set
          ringback: @session.ringback ? '%(3000,3000,437,1317)'
          instant_ringback: true

"Tonalité d'acheminement", for nostalgia's sake.

        @action 'playback', 'tone_stream://%(50,50,437,1317);loops=-1'

Options might be provided for either `place-call` or `call-to-conference`.
They are used in `tough-rate/middleware/call-handler`.

      options = await @reference.get_call_options()
      if options?
        @session.call_options = options

      debug 'Ready', {destination,account,options}
