---
layout: post
title:  "How to Implement dynamic Subdomains in a Ruby on Rails App Hosted on Heroku"
excerpts: ""
---

In the digital age, many modern SaaS platforms provide each user or company with a dedicated subdomain (e.g. `company.myapp.com`) that they can share with clients. This article will guide you through setting up similar functionality for your Ruby on Rails application hosted on Heroku.

## 1. Prerequisites

Before diving into the setup process, it's important to ensure a few foundational elements are in place to support subdomains in your application:

- **Custom Domain**: Your application should be running on a custom domain as Heroku does not support subdomains for *.herokuapp.com URLs.
- **Dedicated Staging**: It's advisable to use a dedicated domain for staging and review apps rather than subdomains of your production URL (`staging.myapp.com`) to simplify management and avoid conflicts.
- **Setup on Heroku**: Ensure that your Heroku custom domains are correctly set up with wildcard domains. This configuration enables Heroku to recognize requests to various subdomains of your application.
- **SSL Certificate**: Since Heroku does not manage wildcard SSL certificates, you will need to handle this aspect independently. I use Cloudflare to manage SSL and have uploaded an Origin Certificate to Heroku, valid for 15 years. For more information on how to do this, refer to the [Heroku SSL documentation](https://help.heroku.com/GVS2BTB5/why-am-i-getting-error-525-ssl-handshake-failed-with-cloudflare-when-using-a-herokudns-com-endpoint).

![Heroku custom domain and SSL settings](/assets/custom-subdomain/heroku-domains.png)

## 2. Adding subdomains to our model

For this example, our application will manage two types of subdomains:

- **app.**: This will include all core functionalities of our application.
- **organization_subdomain.**: This represents the public URL for each organization

First, we need to add a subdomain attribute to our Organization model. To do this, generate a migration as follows:

```bash
rails g migration addSubdomainToOrganization subdomain
```
This command creates a new migration file that adds a subdomain column to the Organizations table in your database.

Next, we implement validations to ensure subdomain uniqueness and exclude reserved names.

```ruby
# app/models/organization.rb

class Organization < ApplicationRecord
  # ...

  EXCLUDED_SUBDOMAINS = %w[feedback contact click cdn www mail email ftp admin support blog api dev test staging internal secure app static media].freeze

  validates :subdomain, uniqueness: true, presence: true,
            format: { with: /\A[a-z0-9][a-z0-9-]*\z/ },
            exclusion: { in: EXCLUDED_SUBDOMAINS },
            length: { minimum: 5, maximum: 40 }

  validate :subdomain_cannot_change, on: :update

  private

  def subdomain_cannot_change
    if will_save_change_to_subdomain? && subdomain.present?
      errors.add(:subdomain, :cannot_change)
    end
  end
end
```

**Important Considerations:**

- **DNS Settings**: Review your DNS settings to ensure there are no conflicts with the subdomains you intend to use.
- **Subdomain Changes**: To simplify management and avoid potential SEO and user experience issues, we restrict the ability to change subdomains after they are created.


## 3. Creating Routing Constraints

Routing constraints in Rails are powerful tools that help us manage traffic based on specific conditions such as subdomains. This setup is crucial for directing users to the appropriate part of your application based on the subdomain they access.

We need to define two key constraints:
- **AppSubdomain**: This constraint handles traffic for `app.*` subdomain
- **OrganizationSubdomain**: This constraint handles traffic for `organization_subdomain.*` subdomain

```ruby
# lib/constraints/organization_subdomain.rb

class OrganizationSubdomain
  def matches?(request)
    Organization.find_by("subdomain = ?", request.subdomain).present?
  end
end
```


```ruby
# lib/constraints/app_subdomain.rb

class AppSubdomain
  def matches?(request)
    app_subdomain?(request) || heroku_host?(request) || localhost?(request) || Rails.env.test?
  end

  private

  def app_subdomain?(request)
    request.subdomain == "app"
  end

  def heroku_host?(request)
    return false unless %w[production staging].include?(Rails.env)

    authorized_host = case Rails.env
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
end
```

This constraint ensures that the core application functionalities are accessible through the `app` subdomain. It also accommodates traffic from Heroku-hosted environments (like `staging` and `review apps`) and allows for `local` development access.

With these constraints defined, we can now apply them within our `routes.rb` file to manage the application's routing logic effectively:

```ruby
# config/routes.rb

require "constraints/app_subdomain"
require "constraints/organization_subdomain"

Rails.application.routes.draw do
  constraints(AppSubdomain.new) do
    # Routes for core application functionalities
  end

  constraints(OrganizationSubdomain.new) do
    # Routes for individual organization subdomains
  end
end
```

**Recommendation**: To maintain clean and organized routing, it's advisable to namespace all routes dedicated to subdomains in `subdomains/***_controllers.rb`.

## 4. Additionnal configuration

After setting up routing constraints, the next step is to configure your application for **local development** and **session management**. This ensures that your development environment mimics production settings and that sessions are handled correctly across all subdomains.

### Local Development Setup

To accommodate subdomains on your local development server, you must adjust the top-level domain (TLD) length in your environment configurations. This setting is crucial for correctly parsing subdomains from the URL.

```ruby
# config/environments/development.rb

config.action_dispatch.tld_length = 0
```

Setting tld_length to 0 allows you to access subdomains like `http://app.localhost:3000` without additional domain setup, simplifying local development.

### Session Management Across Subdomains (Devise)

When using Devise or another session-based authentication framework, it’s important to configure the session store to properly manage subdomains. This setup prevents session cookies from being shared across all subdomains by default, enhancing security.

```ruby
# config/application.rb

config.session_store :cookie_store, key: "_hubflo_session", tld_length: 0
```

### Subdomains with Heroku Review apps

Managing subdomains in Heroku review apps involves dynamically creating DNS records. This process is crucial for ensuring that each review app has its subdomain properly set up and routed.

Here’s how to implement this in a post-deploy script:

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

Configuration in `app.json`:

```js
// app.json

{
  "environments": {
    "review": {
      "scripts": {
        "postdeploy": "bundle exec rake reviewapp:dns_setup",
        "pr-predestroy": "bundle exec rake reviewapp:dns_destroy"
      }
    }
  }
  // ...
}
```

These scripts ensure that DNS settings for each review app are configured upon deployment and properly removed when the app is deleted. This automation helps maintain a clean and consistent environment across all instances.

## 5. Conclusion

I hope this article has helped you integrate custom subdomains into your Ruby on Rails applications on Heroku! Every project has its unique challenges and requirements, so remember to adapt these strategies to fit your specific application needs and make them work for you. Good luck with your projects ! 😀
