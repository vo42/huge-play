    pkg = require '../../../package'
    @name = "#{pkg.name}/middleware/carrier/egress/post"
    seem = require 'seem'
    @include = seem ->
      return unless @session.direction is 'ingress'

      yield @set
        ccnq_direction: @session.direction
        ccnq_profile: @session.profile
        ccnq_from_e164: @source
        ccnq_to_e164: @destination
        sip_cid_type: 'pid'
        progress_timeout: 12
        call_timeout: 300
        t38_passthru: true

      yield @export
        sip_wait_for_aleg_ack: true
        t38_passthru: true

      return