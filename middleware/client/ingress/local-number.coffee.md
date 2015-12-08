    seem = require 'seem'
    pkg = require '../../../package.json'
    @name = "#{pkg.name}:middleware:client:ingress:local-number"
    debug = (require 'debug') @name

    @include = seem ->

      return unless @session.direction is 'ingress'

Global number provides inbound routing
--------------------------------------

Now, we have two cases:
- either the global-number record contains the information we need to do translation (except for the source, which is why this module needs to be inserted between the national translations, and the Centrex routing modules), and we use that information;
- or it doesn't and we use a specific middleware to do the translation (based on the destination number, typically, to put that number in a specific `national` dialplan). The `france` module in the current directory illustrates this.

      return unless @session.e164_number?.local_number?

These are used e.g. for Centrex, and override the destination number and number-domain.

Note: we still keep going through the national modules because we need source number translation from `e164` to `national`.


      debug 'Using local_number'

The dialplan and country (and other parameters) might also be available in the `number_domain:` record and should be loaded from there if the global-number does not specify them.

      [number,number_domain] = @session.e164_number.local_number.split '@'
      @destination = number
      @session.number_domain = number_domain

      @session.number_domain_data = yield @cfg.prov
        .get "number_domain:#{number_domain}"
        .catch (error) =>
          debug "number_domain #{number_domain}: #{error}"
          {}

      @session.dialplan = @session.number_domain_data?.dialplan ? @session.e164_number.dialplan
      @session.country  = @session.number_domain_data?.country  ? @session.e164_number.country