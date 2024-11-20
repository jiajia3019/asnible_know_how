ansible-serverspec-project/
├── ansible/
│   ├── playbooks/
│   │   └── site.yml
│   ├── roles/
│   │   ├── webserver/
│   │   │   ├── tasks/
│   │   │   │   └── main.yml
│   │   │   └── ...
│   │   └── database/
│   │       ├── tasks/
│   │       │   └── main.yml
│   │       └── ...
│   ├── inventories/
│   │   ├── production
│   │   └── staging
│   └── ...
├── serverspec/
│   ├── spec/
│   │   ├── data/
│   │   │   ├── variables.yml
│   │   │   ├── production.yml
│   │   │   └── staging.yml
│   │   ├── helpers/
│   │   │   └── variables.rb
│   │   ├── spec_helper.rb
│   │   ├── webserver_spec.rb
│   │   ├── database_spec.rb
│   │   └── ...
│   ├── Rakefile
│   ├── Gemfile
│   └── README.md
├── ci/
│   └── ci-config.yml
├── README.md
└── ...