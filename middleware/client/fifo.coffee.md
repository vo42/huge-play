    pkg = require '../../package'
    @name = "#{pkg.name}:middleware:client:fifo"
    debug = (require 'tangible') @name
    qs = require 'querystring'

    @description = '''
      Handles routing to a given queue/ACD/hunt-group.
    '''

    sleep = (timeout) ->
      new Promise (resolve) ->
        setTimeout resolve, timeout

    second = 1000

    @include = ->

FIFO handling
=============

      debug 'Starting'

      return unless @session?.direction is 'fifo'

      fifo_uri = (id,name) =>
        @prompt.uri 'prov', 'prov', id, name

      unless @session.fifo?
        debug 'Missing FIFO data'
        return

      fifo = @session.fifo

Build the full fifo name (used inside FreeSwitch) from the short fifo-name and the number-domain.

      @notify state: 'fifo', fifo: fifo.full_name

Ready to send, answer the call.

      debug 'Answer'
      await @action 'answer'
      @session.sip_wait_for_aleg_ack = false

      await @export
        t38_passthru: false

Basically if the pre_answer we should wait; once the call is answered we won't be getting any more ACK, though.

        sip_wait_for_aleg_ack: @session.sip_wait_for_aleg_ack

* session.fifo.announce (string) Name of the FIFO announce file (attachment to the doc:number_domain document).
* session.fifo.music (string) Name of the FIFO music file (attachment to the doc:number_domain document).

      id = "number_domain:#{@session.number_domain}"

      debug 'Send to FIFO'
      await @set
        continue_on_fail: true

      if @session.ringback?
        announce_uri = @session.ringback
      if @session.announce?
        announce_uri = @session.announce
      if fifo.announce?
        announce_uri = fifo_uri id, fifo.announce
      if announce_uri?
        await @set ringback: announce_uri

      if @session.music?
        music_uri = @session.music
      if fifo.music?
        music_uri = fifo_uri id, fifo.music
      if music_uri?
        await @export hold_music: music_uri

FIXME: This is taken from the centrex-{country} code, but really it should be more generic.

      if @session.ccnq_from_e164?
        @source = "+#{@session.ccnq_from_e164}"

      if fifo.tags?
        await @user_tags fifo.tags

      if fifo.required_skills?
        for skill in fifo.required_skills
          await @reference.add_ 'skill', skill

      if typeof fifo.queue is 'string'
        await @reference.add_ 'queue', fifo.queue

      if fifo.priority?
        await @reference.set 'priority', fifo.priority

      if fifo.broadcast
        await @reference.set 'broadcast', true

Call-center
===========

If the call-group should use the queuer, then do that.

      if fifo.queue

        @notify state:'queue', name:fifo.full_name, queue: fifo.queue

        {queuer} = @cfg
        call = await @queuer_call()

Map "reference" (i.e. blue-rings) "tags" (i.e. sets or values) to black-metal / normal-key "tags" (i.e. an array of strings).

        skills_tags = (await @reference.all_ 'skill').map (s) -> "skill:#{s}"
        queues_tags = (await @reference.all_ 'queue').map (q) -> "queue:#{q}"
        priority  = await @reference.get 'priority'
        broadcast = switch await @reference.get 'broadcast'
          when true, 'true'
            true
          else
            false
        ref_tags = [
          skills_tags...
          queues_tags...
        ]
        ref_tags.push "priority:#{priority}" if priority?
        ref_tags.push 'broadcast' if broadcast

        await call.set_tags ref_tags

        if music_uri?
          await call.set_music music_uri

        if announce_uri?
          @action 'endless_playback', announce_uri # async

        await call.set_remote_number @source
        await call.set_alert_info @session.alert_info if @session.alert_info?
        await call.set_domain @session.number_domain
        await call.clear()
        await queuer.queue_ingress_call call

If the call is not processed (no agents are ready), attemp overflow.

Overflow is the weird concept that instead of giving a caller access to all of our capable
agents immediately, we decide to lengthen response time and leave some agents idle, by only
allowing access to them if some conditions are met.
Since we are trying to extend the pool of agents, this is only possible by adding more
desirable queues to a given call. (Adding more required skills would build a smaller pool.)

        if queues_tags.length is 0
          debug 'no queues, no overflow'
          return

        ingress_pool = queuer.ingress_pool @session.number_domain

        attempt_overflow = (suffix) =>
          debug 'attempt overflow', queues_tags, suffix
          if await ingress_pool.has call
            ok = false
            # call.has_tag() and call.add_tag() from black-metal / normal-key
            for tag in queues_tags when await call.has_tag tag
              await call.add_tag "#{tag}:#{suffix}"
              ok = true
            ok
          else
            false

Attempt overflow immediately

        unless attempt_overflow 'overflow'
          return

Attempt overflow after a delay

        {max_overflow} = @cfg
        max_overflow ?= 5*60
        {overflow_interval} = @cfg
        overflow_interval ?= 30

        delay = 0
        while delay <= max_overflow
          await sleep overflow_interval*second
          delay += overflow_interval
          unless await attempt_overflow "overflow:#{delay}s"
            return

        return

Hunt-group
==========

Otherwise use the hunt-group behavior.

      @notify state:'group', name:fifo.full_name

      sofias = []

* session.fifo.members (array) (required) List of static members for this hunt-group/ACD/FIFO.
* session.fifo.members[].recipient (string) (required) Local-number in the FIFO's number-domain. The recipients are dialed without regards to their CFA, .. settings.
* session.fifo.members[].delay (integer) Number of seconds to wait before trying to call this recipient, in seconds. Zero or `null` means 'call immediately'. Default: 0.
* session.fifo.members[].progress_timeout (integer) Number of seconds before declaring a recipient unreachable (unable to ring the phone). Default: the progress-timeout of the FIFO.
* session.fifo.members[].timeout (integer) Number of seconds before declaring a recipient unreachable (did not answer), in seconds. Zero means 'wait indefinitely'. Default: the timeout of the FIFO.
* session.fifo.timeout (integer) Default number of seconds before declaring a recipient unreachable (how long to let the recipient's phone ring). Default: zero, meaning 'wait indefinitely'.
* session.fifo.progress_timeout (integer) Default number of seconds before declaring a recipient unreachable (unable to ring the phone). Default: 4.

      fifo_timeout = fifo.timeout ? 0
      fifo_progress_timeout = fifo.progress_timeout ? 4

      recipients = []
      for member,i in fifo.members
        # Backward-compatible
        if 'string' is typeof member
          recipient = member
          leg_delay_start = i*5
          leg_progress_timeout = 4
          leg_timeout = 60
        else
          recipient = member.recipient
          leg_delay_start = member.delay ? 0
          leg_progress_timeout = member.progress_timeout ? fifo_progress_timeout
          leg_timeout = member.timeout ? fifo_timeout

        recipients.push "#{recipient}@#{@session.number_domain}"

        sofias.push await @sofia_string recipient, ["#{k}=#{v}" for own k,v of {
          t38_passthru: false
          leg_delay_start
          leg_progress_timeout
          leg_timeout
        }]
      debug 'bridge', sofias
      res = await @action 'bridge', sofias.join ','

      data = res.body
      @session.bridge_data ?= []
      @session.bridge_data.push data
      debug 'Returned from FIFO', data

Marking missed calls
--------------------

`variable_originated_legs: [ 'uuid:name:number', … ]`
`variable_originate_causes: [ 'uuid:cause', … ]`

Some causes:
- `NONE` -- race winner
- `LOSE_RACE` -- race loser
- `ALLOTTED_TIMEOUT` -- all losers
- `NO_ROUTE_DESTINATION` -- technical difficulty
- `ORIGINATOR_CANCEL` -- caller hung up, all losers

FIXME: what is the `cause` when a call was not presented? (esp. if there was a winner)

      causes = data.variable_originate_causes?.map (str) -> str.split(':')[1]
      for cause,i in causes ? []
        do (cause,agent = recipient[i]) =>
          switch cause
            when 'NONE'
              @notify {state: 'hunt-bridging', agent}
              @notify {state: 'answered', agent}
            when 'NO_ROUTE_DESTINATION'
              no
            else
              @notify {state: 'hunt-bridging', agent}
              @notify {event: 'missed', agent}

Detecting transfer
------------------

Available parameters related to transfer are (in the case `bridge` is used):
- `variable_transfer_source` (string)
- `Caller-Transfer-Source` (string)
- `variable_transfer_history` (string or array)

Blind-transfer:
- `variable_transfer_history`: `"1461931135:41606813-d6c0-4bf9-af49-b2017c23ab7c:bl_xfer:endless_playback:http://(nohead=true)127.0.0.1:5714/fifo/number_domain%3Atest.centrex.phone..."`

Attended-transfer:
- `variable_transfer_history`: `[ "1461928571:4edd3fa0-8725-41a4-9322-d1e6b636a913:att_xfer:10@test.centrex.phone../33643482771@178.250.209.67", "1461928571:017ae036-cfdb-4e35-9bd8-0686db3897a0:uuid_br:36093e6f-cc71-484e-8cc7-eb70d42c10be" ]`

The first number in those strings is the timestamp in seconds.
In the case of `uuid_br`, the UUID at the end is the `Other-Leg-Unique-ID`.

      xfer = data.variable_transfer_history
      if xfer?
        debug 'Call was transferred', xfer
        return

      cause = data.variable_originate_disposition

      debug "FIFO returned with cause #{cause}"

      if cause in ['NORMAL_CALL_CLEARING', 'SUCCESS', 'NORMAL_CLEARING']
        debug "Successful call when routing FIFO #{fifo.full_name} through #{sofias.join ','}"
        await @action 'hangup'
        return

      if cause is 'ORIGINATOR_CANCEL'
        return

* session.fifo.voicemail (string) If present, the call is redirected to this number's voicemail box if the FIFO failed (for example because no agents are available).
* session.fifo.user_database (string) If present, the call is redirected to this voicemail box if the FIFO failed (for example because no agents are available). Default: use session.fifo.voicemail if present.

      if fifo.voicemail?
        debug 'Send to voicemail'
        @destination = fifo.voicemail
        @direction 'voicemail'
        await @validate_local_number()
        return

      if fifo.user_database?
        debug 'Send to voicemail (user-database)'
        @destination = 'user-database'
        @session.voicemail_user_database = fifo.user_database
        @session.voicemail_user_id = fifo.full_name
        @direction 'voicemail'
        return

      debug 'Hangup'
      await @action 'hangup'
