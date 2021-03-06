    pkg = require '../../../package.json'
    @name = "#{pkg.name}:middleware:client:ingress:centrex-FR"
    debug = (require 'tangible') @name
    tones = require '../tones'

    default_internal_ringback = tones.fr.ringback
    default_internal_music = tones.loop tones.fr.waiting

    @include = ->

      return unless @session?.direction is 'ingress'
      return unless @session.dialplan is 'centrex'
      return unless @session.country is 'fr'

Add the outside line prefix so that the call can be placed directly.
Note: since we might also come here because we are routing an internal call, skip if the source doesn't need translation.

      @session.centrex_external_line_prefix ?= '9'

      await @export sip_invite_domain: @session.number_domain

      if @session.centrex_internal
        # await @export alert_info: 'info=centrex-internal'
        await @export alert_info: '<http://127.0.0.1/Bellcore-dr2>'
      else
        if @session.ccnq_from_e164?
          @source = "+#{@session.ccnq_from_e164}"
        else
          @source = "#{@session.centrex_external_line_prefix}#{@source}" if @source[0] is '0'

Also force a normal ringback and no muzak on internal calls.

      if @session.centrex_internal
        @session.ringback = @cfg.internal_ringback ? default_internal_ringback
        @session.music = @cfg.internal_music ? default_internal_music
