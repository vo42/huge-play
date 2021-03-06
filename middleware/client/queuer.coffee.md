    @name = 'huge-play:middleware:client:queuer'
    {debug,foot,heal} = (require 'tangible') @name
    pkg = name:'huge-play'
    Moment = require 'moment-timezone'
    {SUBSCRIBE,UPDATE} = require 'red-rings/operations'

    Solid = require 'solid-gun'
    make_id = ->
      Solid.time() + Solid.uniqueness()

    Queuer = require 'black-metal/queuer'
    request = require 'superagent'

    compile = require 'flat-ornament/compile'

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    domain_of = (key) ->
      key?.split('@')[1]

    now = (tz = 'UTC') ->
      Moment().tz(tz).format()

    {TaggedCall,TaggedAgent} = require 'black-metal/tagged'
    monitor = require 'screeching-eggs/monitor'
    FS = require 'esl'
    RedisInterface = require 'normal-key/interface'

    SIPSender = require 'five-toes/sip-sender'
    yealink_message = require 'five-toes/yealink-message'
    ccnq4_resolve = require 'five-toes/ccnq4-resolve'
    dgram = require 'dgram'

    Nimble = require 'nimble-direction'
    CouchDB = require 'most-couchdb'

    socket = dgram.createSocket 'udp4'
    sender = new SIPSender socket

    @end = ->
      socket.close()

    @server_pre = ->

      cfg = @cfg

      HugePlayReference = @cfg.Reference
      {api,host} = @cfg
      prov = new CouchDB (Nimble cfg).provisioning
      profile = @cfg.session?.profile
      p = @cfg.profiles?[profile]
      if p?
        port = p.egress_sip_port ? p.sip_port+10000

      unless cfg.local_redis_client? and prov? and profile? and host? and port? and api? and HugePlayReference?
        debug.dev 'Missing configuration'
        return

How long should we keep the state of a call after the last update?

      call_timeout = 8*3600

How long should we keep the state of an agent after the last update?

      agent_timeout = 12*3600

HugePlayCall
------------

      class HugePlayCall extends TaggedCall

        interface: new RedisInterface cfg.local_redis_client, call_timeout
        __api: api

        profile: "#{pkg.name}-#{profile}-egress"
        Reference: HugePlayReference

        build_notification: (data) ->
          notification =
            report_type: 'queuer'
            host: host
            now: Date.now()

            domain: await @get_domain().catch -> null
            key: @key
            id: await @get_id().catch -> null

            call_state: await @state().catch -> null
            remote_number: await @get_remote_number().catch -> null
            alert_info: await @get_alert_info().catch -> null
            reference: await @get_reference().catch -> null
            answered: switch await @answered().catch -> null
              when 'true', true
                true
              when 'false', false
                false
              else
                null
            presenting: await @count().catch -> null
            tags: await @tags().catch -> []  # agent.tags() in black-metal / normal-key

          for own k,v of data when v?
            switch k
              when 'agent', 'call', 'agent_call', 'remote_call'
                v = v.key if typeof v isnt 'string'
            notification[k] ?= v

          notification

        notify: (data) ->
          debug 'call.notify', data
          notification = await @build_notification data

          if notification.domain?
            cfg.rr.notify "number_domain:#{notification.domain}", "call:#{notification.id}", notification

          debug 'call.notify: send', notification
          notification

HugePlayAgent
-------------

      resolve = ccnq4_resolve Nimble cfg

      class HugePlayAgent extends TaggedAgent

        interface: new RedisInterface cfg.local_redis_client, agent_timeout

        notify: (data) ->
          debug 'agent.notify', @key, data

          {old_state,state,event,reason} = data

          notification =
            report_type: 'queuer'
            host: host
            now: Date.now()

            state: state
            old_state: old_state
            event: event
            reason: reason
            agent: @key
            number: @number
            number_domain: @domain
            missed: await @get_missed().catch -> 0
            count: await @count().catch -> 0

          notification.tags = await @tags().catch -> []  # agent.tags() in black-metal / normal-key
          agent_name = await (@get 'name').catch -> null
          if agent_name?
            notification.agent_name = agent_name

          offhook = await @get_offhook_call().catch -> null
          if offhook
            notification.offhook = true
          else
            notification.offhook = false

          for own k,v of data when typeof v in ['number','string','boolean']
            notification[k] ?= v

Module `black-metal` 8.3.0 will report the call object as `data.call`.

If `data.call` is present we notify using the call's process; if it isn't we notify directly.
This avoids sending two messages for the same event (one with incomplete data, the other with complete data).

          if data.call?
            notification = await data.call.notify notification

          cfg.rr.notify "agent:#{notification.agent}", "agent:#{notification.agent}", notification

          notify = (msg) =>
            content = yealink_message msg
            dest = await resolve @key
            await sender.notify dest, content
            return

          switch state
            when 'logged_out'
              await notify 'Logout'
            when 'wrap_up'
              await notify 'Wrap-up'
            when 'idle'
              await notify 'Prêt'

          debug 'agent.notify: done', @key, notification
          return

        create_egress_call: (body) ->
          debug 'create_egress_call', @domain, body

          @number_domain_data ?= await prov
            .get "number_domain:#{@domain}"
            .catch (error) -> null

          unless @number_domain_data?
            debug 'create_egress_call: missing number-domain', @domain
            return

          {account,queuer_webhook,timezone} = @number_domain_data

          timezone ?= null

          await @set 'timezone', timezone

          unless account?
            debug 'create_egress_call: no account', @domain
            return null

          if not body?
            unless queuer_webhook?
              debug 'create_egress_call: no queuer_webhook', @domain
              return null

            tags = await @tags()  # agent.tags() in black-metal / normal-key
            options = {@key,@number,@domain,tags}
            debug 'create_egress_call: send request', options
            {body} = await request
              .post queuer_webhook
              .send options

          body.tags ?= []
          unless body.destination? and body.tags? and body.tags.length?
            debug 'create_egress_call: incomplete response', @domain, body
            return null

          body.tags.push 'queuer'

          debug 'create_egress_call: creating call', @domain, body
          body.tags.push 'egress'

See `in_domain` in black-metal/tagged.

          body.tags.push "number_domain:#{@domain}"

          endpoint = @key
          reference = new HugePlayReference()
          _id = reference.id

FIXME This is highly inefficient, we should be able to create the structure at once.

          await reference.set_endpoint endpoint
          await reference.set_account account
          await reference.set_number_domain @domain
          await reference.set_number endpoint # Centrex-only
          await reference.set_destination body.destination
          await reference.set_source @number
          await reference.set_domain "#{host}:#{port}"
          await reference.set_block_dtmf true

This is a "fake" call-data entry, to record the data we used to trigger the call for call-reporting purposes.

          call_data =
            uuid: 'create-egress-call'
            session: "#{_id}-create-egress-call"
            reference: _id
            start_time: now timezone
            source: @number
            destination: body.destination
            timezone: timezone

            timestamp: now timezone
            host: host
            type: 'call'

            event: 'create-egress-call'

Notice that for the new call we explicitely do _not_ `set_id` since this is how `originate_external` recognizes that the call needs to be set up.

          call = new HugePlayCall make_id()
          await call.set_domain @domain
          await call.set_started_at()
          await call.set_reference _id
          await call.set_remote_number body.destination
          await call.set_tags body.tags
          await call.set 'timezone', timezone

          # async
          heal @notify call_data

          debug 'create_egress_call: complete'

          return call

Queuer
------

The queuer's Redis is used for call pools and the agents pool.
Since we're bound to a server for domains it's OK to use the local Redis.

      pools_redis_interface = new RedisInterface cfg.local_redis_client, agent_timeout

      class HugePlayQueuer extends Queuer pools_redis_interface, Agent: HugePlayAgent, Call: HugePlayCall
        notify: (key,id,data) ->
          notification =
            report_type: 'queuer'
            host: host
            now: Date.now()

          for own k,v of data when typeof v in ['number','string','boolean']
            notification[k] ?= v

          if data.call?
            notification = await data.call.notify notification
          cfg.rr.notify key, id, notification
          return

      @cfg.queuer = new HugePlayQueuer @cfg

      options =
        host: @cfg.socket_host ? '127.0.0.1'
        port: @cfg.socket_port ? 5722
      client = FS.createClient options
      monitor client, @cfg.queuer.Call

RedRings
--------

RedRings for agents:

      prov = new CouchDB (Nimble cfg).provisioning

      cfg.rr
      .receive 'agent:*'
      .forEach (msg) ->
        switch
          when msg.op is SUBSCRIBE
            # get agent state
            return unless $ = msg.key?.match /^agent:(\S+)$/
            key = $[1]
            is_remote = await cfg.is_remote domain_of key
            return if is_remote isnt false

            debug 'queuer:get-agent-state', key

            agent = new HugePlayAgent key
            state = await agent.state().catch -> null
            await agent.notify {state}

          when msg.op is UPDATE
            # log agent in or out; use redring.create() for login, redring.delete() for logout

            return unless $ = msg.id?.match /^agent:(\S+)$/
            key = $[1]
            is_remote = await cfg.is_remote domain_of key
            return if is_remote isnt false

            if msg.deleted

              debug 'queue:log-agent-out', key

              agent = new HugePlayAgent key
              await agent.clear_tags() # in normal-key
              await agent.transition 'logout'

            else

              debug 'queue:log-agent-in', key

              tags = []
              {skills,queues,broadcast,timezone} = await prov.get "number:#{key}"
              if skills?
                for skill in skills
                  tags.push "skill:#{skill}"
              if queues?
                for queue in queues
                  tags.push "queue:#{queue}"
              if broadcast
                tags.push 'broadcast'

              agent = new HugePlayAgent key
              await agent.add_tags tags  # in normal-key

              ### FIXME FIXME
              if ornaments?
                ctx = {agent,timezone}
                fun = compile ornaments
                await fun.call ctx if fun?
              ###

              await agent.accept_onhook()

        return

Note: RedRings agent notifications are handled above (in the HugePlayCall and HugePlayAgent classes).

RedRings for pools:

      cfg.rr
      .receive 'pool:*'
      .filter ({op}) -> op is SUBSCRIBE
      .forEach (msg) ->

        return unless $ = msg.key?.match /^pool:(\S+):(ingress|egress)$/

        domain = $[1]
        name = $[2]

        is_remote = await cfg.is_remote domain
        return if is_remote isnt false

        pool = switch name
          when 'ingress'
            cfg.queuer.ingress_pool domain

          when 'egress'
            cfg.queuer.egress_pool domain

        calls = await pool.calls()
        result = await Promise.all calls.map (call) -> call.build_notification {}

        notification =
          host: host
          now: Date.now()
          calls: result

        cfg.rr.notify msg.key, "number_domain:#{domain}", notification

        return

      return

Middleware
==========

    @include = ->

      if await @reference.get_block_dtmf()
        await @action 'block_dtmf'

      return unless @cfg.queuer?

      {Agent,Call} = @cfg.queuer
      return unless Agent? and Call?

Queuer Call object
------------------

      {uuid} = @call

      @queuer_call = (id) ->
        domain = @session.number_domain
        id ?= uuid
        debug 'queuer_call', id, domain
        queuer_call = new Call id
        await queuer_call.set_domain domain
        await queuer_call.set_started_at()
        await queuer_call.set_id id

        await queuer_call.set_reference @session.reference
        queuer_call

Agent state monitoring
----------------------

      local_server = [@session.local_server,@session.client_server].join '/'

On-hook agent
-------------

      @queuer_login = (source,name,tags,ornaments) ->
        tags ?= []
        debug 'queuer_login', source
        agent = new Agent source
        await agent.set 'name', name
        # await agent.clear_tags()  # in normal-key
        await agent.add_tags tags  # in normal-key

        if ornaments?
          @agent = agent
          fun = compile ornaments, @ornaments_commands
          await fun.call this if fun?
          delete @agent

        await agent.accept_onhook()
        await @report {state:'queuer-login',source,tags}
        agent

      @queuer_logout = (source) ->
        debug 'queuer_logout', source
        agent = new Agent source
        await agent.clear_tags()  # in normal-key
        await agent.transition 'logout'
        await @report {state:'queuer-logout',source}
        agent

Off-hook agent
--------------

      @queuer_offhook = (source,name,{uuid},tags,ornaments) ->
        tags ?= []
        debug 'queuer_offhook', source, uuid
        agent = new Agent source
        await agent.set 'name', name
        await agent.clear_tags()  # in normal-key
        await agent.add_tags tags  # in normal-key

        if ornaments?
          @agent = agent
          fun = compile ornaments, @ornaments_commands
          await fun.call this if fun?
          delete @agent

        call = await agent.accept_offhook uuid
        return unless call?
        await @report {state:'queuer-offhook',source,tags}
        agent

More states
-----------

      resolve = ccnq4_resolve Nimble @cfg

      @queuer_wrapup_complete = (source) ->
        debug 'queuer_wrapup_complete', source
        agent = new Agent source
        await agent.transition 'complete'
        return

      @queuer_pause = (source) ->
        debug 'queuer_pause', source
        agent = new Agent source

        state = await agent.state()
        debug 'queuer_pause: state', source, state

        notify = (msg) ->
          content = yealink_message msg
          dest = await resolve source
          await sender.notify dest, content
          return

        start_pause = =>
          debug 'queuer_pause: start', source
          await agent.transition 'logout', reason: 'pause'

FIXME: pause duration should be set-able by domain / agent

          if @cfg.pause_timeout? && @cfg.pause_timeout > 0
            timeout = @cfg.pause_timeout
            heal =>
              while timeout > 0
                await notify "En pause (#{timeout}s)"
                await sleep 1000
                timeout--
              await end_pause()
              return
          else
            await notify "En pause"
          return

        end_pause = =>
          debug 'queuer_pause: end', source
          await agent.accept_onhook()
          await notify "Pause finie"
          return

        switch state
          when 'logged_out'
            await end_pause()
            return 'end'

          when 'waiting'
            await start_pause()
            return 'start'

          else
            notify "Erreur"

        null

      return
