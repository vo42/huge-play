    pkg = require '../../../package.json'
    @name = "#{pkg.name}:middleware:client:ingress:local-number"
    debug = (require 'tangible') @name
    Nimble = require 'nimble-direction'
    CouchDB = require 'most-couchdb'

    @include = ->

      return unless @session?.direction is 'ingress'

      music_uri = (doc) =>
        return null unless doc.music?
        @prompt.uri 'prov', 'prov', doc._id, doc.music

Global number provides inbound routing
--------------------------------------

Now, we have two cases:
- either the global-number record contains the information we need to do translation (except for the source, which is why this module needs to be inserted between the national translations, and the Centrex routing modules), and we use that information;
- or it doesn't and we use a specific middleware to do the translation (based on the destination number, typically, to put that number in a specific `national` dialplan). The `france` module in the current directory illustrates this.

* doc.global_number.local_number (string) The identifier of the local-number into which this global-number gets translated for inbound calls. (The identifier must have the format `<number>@<number-domain>` and a `number:<number>@<number-domain>` record must exist.)

      return unless @session.e164_number?.local_number?
      await @reference.set_number @session.e164_number.local_number

These are used e.g. for Centrex, and override the destination number and number-domain.

Note: we still keep going through the national modules because we need source number translation from `e164` to `national`.


      debug 'Using local_number', @session.e164_number.local_number

The dialplan and country (and other parameters) might also be available in the `number_domain:` record and should be loaded from there if the global-number does not specify them.

* session.number_domain (string) The number-domain of the current destination.
* session.number_domain_data (object) If present, the content of the `number_domain:<number-domain>` record for the current `session.number_domain`.
* doc.number_domain (object) Record describing a number-domain.
* doc.number_domain._id (required) `number_domain:<number-domain>`
* doc.number_domain.type (required) `number_domain`
* doc.number_domain.number_domain (required) `<number-domain>`

      [number,number_domain] = @session.e164_number.local_number.split '@'
      @destination = number

      @session.number_domain = number_domain
      await @reference.set_number_domain number_domain

      prov = new CouchDB (Nimble @cfg).provisioning
      @session.number_domain_data = await prov
        .get "number_domain:#{number_domain}"
        .catch (error) =>
          debug "number_domain #{number_domain}: #{error.stack ? error}"
          {}

      await @user_tags @session.number_domain_data.tags

* doc.number_domain.dialplan (optional) dialplan used for ingress calls to this domain.
* doc.number_domain.country (optional) country used for ingress calls to this domain.

      @session.timezone = @session.number_domain_data.timezone ? @session.e164_number.timezone
      @session.music    = music_uri @session.number_domain_data
      @session.music   ?= music_uri @session.e164_number
      @session.dialplan = @session.number_domain_data.dialplan ? @session.e164_number.dialplan
      @session.country  = @session.number_domain_data.country  ? @session.e164_number.country
      if @session.country?
        @session.country = @session.country.toLowerCase()

      @report
        state:'local-number'
        number: @session.e164_number.local_number
      debug 'OK'
      return
