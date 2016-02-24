    pkg = require '../../../package.json'
    @name = "#{pkg.name}:middleware:client:egress:privacy-CH"
    debug = (require 'debug') @name
    @include = ->

      return unless @session.direction is 'egress'
      return unless @session.dialplan is 'national'
      return unless @session.country is 'ch'

      debug 'Matching', @destination

      if m = @destination.match /^\*31(\d+)$/
        @destination = m[1]
        debug 'Replacing with ', @destination

Add a `Privacy: id` header.

        @action 'privacy', 'number'