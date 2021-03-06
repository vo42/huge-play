This module should be called before 'local/carrier-ingress' and before 'client-sbc/$${profile_type}-ingress'

    pkg = require '../../../package.json'
    @name = "#{pkg.name}:middleware:client:ingress:pre"
    debug = (require 'tangible') @name

    Nimble = require 'nimble-direction'
    CouchDB = require 'most-couchdb'

    @include = ->
      return unless @session?.direction is 'ingress'

      await @reference.set_endpoint null

Do not process here if the dialplan is already known (e.g. because Centrex sent us here).

      if @session.dialplan?
        debug 'Dialplan already set, skipping'
        return

      debug 'Ready'

Global number
-------------

We retrieve the *global-number* record based on the destination.

* session.e164_number (object) The doc.global_number record for of the destination of an inbound call.

      prov = new CouchDB (Nimble @cfg).provisioning

      @session.e164_number = await prov
        .get "number:#{@destination}"
        .catch -> null
      return unless @session.e164_number?

E.164
-----

All external ingress calls come in as E.164 (without plus sign).

      @session.dialplan = 'e164'
      @session.ccnq_from_e164 = @source
      @session.ccnq_to_e164 = @destination

      await @reference.set_number @destination
      await @user_tags @session.e164_number.tags

The global number might contain additional FreeSwitch variables. Load these extra variables from the record.

* session.e164_number.fs_variables See doc.global_number.fs_variables
* doc.global_number (object, optional) Additional FreeSwitch variables to be set on ingress calls (for the called number). These will show up in CDRs on the client side.

      if @session.e164_number.fs_variables?
        debug 'Using fs_variables'
        await @set @session.e164_number.fs_variables

* doc.global_number.trace (boolean) trace

      if @session.e164_number.trace
        @session.dev_logger = true

* session.e164_number.voicemail_main See doc.global_number.voicemail_main
* doc.global_number.voicemail_main (boolean) If true, the number is the main number for access to voicemail (from an external number).
* session.e164_number.language See doc.global_number.language
* doc.global_number.language (string) Language-code to use for features, e.g. voicemail.

      if @session.e164_number.voicemail_main
        debug 'Using voicemail_main'
        @destination = 'main'
        @session.language = @session.e164_number.language

* doc.global_number.voicemail_number_domain (string) the number-domain for access to voicemail (from an external number).

        @session.number_domain = @session.e164_number.voicemail_number_domain
        await @reference.set_number_domain @session.number_domain
        @direction 'voicemail'

      @notify state:'ingress', e164_number: @session.ccnq_to_e164
      debug 'OK'
      return
