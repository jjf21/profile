All modern SaaS give each user/company a public, dedicated subdomain that they can share to client, for example my_company.mysite.com. 
Let's see how to build this with Ruby on Rails with your app hosted on Heroku.

1. Pre requestiste
You must be running your app on a custom domain, for sure heroku won't let you have subdomain on your url (*.herokuap.com).
Prefer moving your staging from staging.mysite.com if it's your case.
Heroku custom domains should be correctly setup:
  In your custom domains, you must have link the wildcard domains so that Heroku will find you application.
  [Photo of custom domains]
  Heroku don't manage wildcard SSL certificate, so you will need to manage your certificate.
  Personnaly Cloudflare manage our SSL, and I just uploaded an Origin Certificate to Heroku (duration 15 years) (link to doc)

3. Handling routes
For our example we will have two routes constraints on subdomains:
- "app" which will include all the core of our application
- "organization subdomain" which will be the organization public url

Lets add a subdomain attribute to our Organization
```ruby
rails g migration addSubdomainToOrganization subdomain
```

We then need to add some validation on this subdomains

```ruby
# app/models/organization.rb

class Organization < ApplicationRecord
  # ...

  EXCLUDED_SUBDOMAIN = %w[feedback contact click cdn www mail email ftp admin support blog api dev test staging internal secure app static media].freeze

  validates :subdomain,
            uniqueness: true,
            format:     { with: /\A[a-z0-9][a-z0-9-]*\z/ },
            exclusion:  { in: EXCLUDED_SUBDOMAIN },
            length:     { minimum: 5, maximum: 40 }

  validate :subdomain_cannot_change, on: :update

  # ...

  private

  def subdomain_cannot_change
    errors.add(:subdomain, :cannot_change) if will_save_change_to_subdomain? && !can_change_subdomain?
  end
end
```

2 things:
- Go check your DNS to exclude some subdomain you may use.
- I decided to block changing the subdomain because I didn't want to manage redirection

Then we create two constraints

```ruby
# lib/constraints/app_subdomain.rb

class AppSubdomain
  def matches?(request)
    app_subdomain?(request) || heroku_host?(request) || localhost?(request) || Rails.env.test?
  end

  private

  def heroku_host?(request)
    return false unless %w[production staging].include?(Rails.env)

    authorized_host =
      case Rails.env
      when "production"
        "your_app.herokuapp.com"
      when "staging"
        /.*your_app.*.herokuapp.com/ # Here we use regex to handle review apps
      end

    request.host.match(authorized_host)
  end

  def localhost?(request)
    Rails.env.development? && request.host == "localhost"
  end

  def app_subdomain?(request)
    request.subdomain == "app"
  end
end

```

EXPLAIN

```ruby
# lib/constraints/organization_subdomain.rb

class OrganizationSubdomain
  def matches?(request)
    Organization.find_by("subdomain = ?", request.subdomain).present?
  end
end
```

EXPLAIN

We can now apply those constraints to our routes

```ruby
# config/routes.rb

require "constraints/public_portal_subdomain"
require "constraints/app_subdomain"

Rails.application.routes.draw do
  constraints(OrganizationSubdomain.new) do
    # All my subdomains routes
  end

  constraints(AppSubdomain.new) do
    # All my apps routes
  end
end
```

EXPLAIN
Personnaly I decided to namespace all routes dedicated to subdomains in "subdomains/***_controllers.rb"

In local we must adjust the TLD LENGTH, so that we keep `localhost:3000`
```ruby
# config/environment/development.rb

# https://github.com/rails/rails/issues/21582#issuecomment-139539164
# Edit the subdomain level for localhost
config.action_dispatch.tld_length = 0
```

If using Devise
```ruby
    # Define session's cookie for all subdomains
    config.session_store :cookie_store, key: "_myapp_session", tld_length: 0
```

Now you're ready to go.


3. Working with review apps.

To work with subdomain on review apps, we will need to create a DNS record for each review app. We can make it with a postdeploy script

```ruby
namespace :reviewapp do
  desc "Setup Cloudflare DNS record for Heroku Review App"
    task dns_setup: :environment do
      Rails.logger.info("Setup Cloudflare DNS record")
      require "cloudflare"
      require "platform-api"
  
      subdomain = generate_subdomain
      begin
        (heroku_domain, wild_heroku_domain) = create_heroku_custom_domains(subdomain)
        create_cloudflare_dns_records(subdomain, heroku_domain, wild_heroku_domain)
        Rails.logger.info("DNS records successfully added")
      rescue Excon::Error::UnprocessableEntity
        Rails.logger.info("DNS records could not be created")
      end
    end
  
    desc "Remove DNS Record from Cloudflare for Heroku Review App upon deletion"
    task dns_destroy: :environment do
      require "cloudflare"
  
      subdomain = generate_subdomain
      delete_cloudflare_dns_records(subdomain)
    end
  
    private
  
    def generate_subdomain
      "pr-#{ENV['HEROKU_PR_NUMBER']}"
    end
  
    def create_heroku_custom_domains(subdomain)
      # Configure custom domain in Heroku
      heroku_app_name = ENV["HEROKU_APP_NAME"]
      heroku_client = PlatformAPI.connect_oauth ENV["HEROKU_PLATFORM_TOKEN"]
      custom_domain = "staging_url.com".freeze
      wild_subdomain = ["*", subdomain].join(".")
  
      hostname = [subdomain, custom_domain].join(".")
      heroku_client.domain.create(heroku_app_name, hostname: hostname, sni_endpoint: nil)
      heroku_domain = heroku_client.domain.info(heroku_app_name, hostname)["cname"]
  
      wild_hostname = [wild_subdomain, custom_domain].join(".")
      heroku_client.domain.create(heroku_app_name, hostname: wild_hostname, sni_endpoint: nil)
      wild_heroku_domain = heroku_client.domain.info(heroku_app_name, wild_hostname)["cname"]
  
      [heroku_domain, wild_heroku_domain]
    end
  
    def create_cloudflare_dns_records(subdomain, heroku_domain, wild_heroku_domain)
      wild_subdomain = ["*", subdomain].join(".")
  
      Cloudflare.connect(token: ENV["CLOUDFLARE_KEY"]) do |connection|
        zone = connection.zones.find_by_id(ENV["CLOUDFLARE_ZONE_ID"])
  
        dns_record = zone.dns_records.find_by_name(subdomain)
        wild_dns_record = zone.dns_records.find_by_name(wild_subdomain)
  
        zone.dns_records.create("CNAME", subdomain, heroku_domain, proxied: false) if dns_record.nil?
        zone.dns_records.create("CNAME", wild_subdomain, wild_heroku_domain, proxied: false) if wild_dns_record.nil?
      end
    end
  
    def delete_cloudflare_dns_records(subdomain)
      custom_domain = "demo-hubflo.com".freeze
      wild_subdomain = ["*", subdomain].join(".")
  
      Cloudflare.connect(token: ENV["CLOUDFLARE_KEY"]) do |connection|
        zone = connection.zones.find_by_id(ENV["CLOUDFLARE_ZONE_ID"])
  
        dns_record = zone.dns_records.find_by_name([subdomain, custom_domain].join("."))
        wild_dns_record = zone.dns_records.find_by_name([wild_subdomain, custom_domain].join("."))
  
        dns_record&.delete
        wild_dns_record&.delete
      end
    end
  end
end
```

