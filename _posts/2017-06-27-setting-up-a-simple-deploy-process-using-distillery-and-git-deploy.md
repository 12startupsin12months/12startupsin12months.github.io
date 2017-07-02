---
layout: post
title:  "Setting up a simple deploy process using distillery and git deploy"
date:   2017-06-27
published: true
comments: true
categories:
  - Elixir
  - Distiller
  - Git Deploy
  - Deploy
---

One of the tools I love from the ruby world is [git-deploy](https://github.com/mislav/git-deploy). It is awesome for what it does.
And, my current setup is a simple single phoenix server. So, using git-deploy makes sense.

Here are the steps to set it up for posterity.

 1. Install git-deploy on your local computer. You need ruby for this to work: `gem install git-deploy`
 2. I am assuming that you have an ubuntu based server for this tutorial and that the name of the app is `sprymesh`. You need to install `esl-erlang`, `elixir` and `nodejs` on the server.

        # install erlang and elixir
        wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb && sudo dpkg -i erlang-solutions_1.0_all.deb
        sudo apt-get update
        # make and build-essential are needed by a few apps
        sudo apt-get install esl-erlang elixir make build-essential
        mix local.hex
        mix local.rebar
        mix local.rebar3
        # install nodejs
        curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -
        sudo apt-get install -y nodejs

 3. Install postgresql on the server, if it is not already present.
 3. Create a user specific account on the server.

        sudo mkdir -p /opt/www
        # name of my user account is sprymesh
        sudo useradd --home /opt/www/sprymesh --create-home --shell /bin/bash sprymesh
        # create the database user
        sudo -u postgres createuser --echo --no-createrole --no-superuser --createdb sprymesh
        # setup the password for the above pg user and tweak the pg_hba.conf if needed
        sudo -u sprymesh createdb sprymesh_prod
        # test by opening a psql connection
        psql --username=sprymesh --password sprymesh_prod

 4. Add distillery to your deps

        {:distillery, "~> 1.4"},

 5. Create a `rel/config.exs`

        mix release.init

 5. Create a git tag with the value 'v1.0'. This is used to build an incrementing version number for our release.

        git tag --annotate --message 'First version' v1.0

 5. Tweak your mix.exs's version

        def project do
          [app: :danny_web,
           version: app_version(),
          # ...
        end

        defp app_version do
          # get git version
          {ts, 0} = System.cmd("git", ~w[describe])
          _git_version = String.strip(ts) |> String.split("-") |> Enum.take(2) |> Enum.join(".") |> String.replace_leading("v", "")
        end

 6. Add your ssh key to `~sprymesh/.ssh/authorized_keys`
 6. Create your git deploy setup

        git remote add prod "sprymesh@sprymesh.com:/opt/www/sprymesh"
        git deploy setup -r prod
        git deploy init

 6. Create a /opt/www/sprymesh/env file containing your env vars/secrets

        MIX_ENV=prod
        SM_APPSIGNAL=....
        DATABASE_URL=ecto://un:pw@localhost:5432/db_prod
        PORT=4000

 6. Change `deploy/before_restart` to something like below

        #!/bin/bash -e

        # source env so that we have all secrets setup
        source <(sed 's/^/export /' ./env)

        # deps are needed
        mix deps.get

        # build the js files
        (cd assets && npm install --no-progress && ./node_modules/brunch/bin/brunch build --production)

        # build release
        mix phoenix.digest
        mix release

 7. Change `deploy/restart` to

        #!/bin/bash

        echo "restarting sprymesh"
        sudo systemctl stop sprymesh.service
        sudo systemctl start sprymesh.service
        echo "finished restarting sprymesh"

 8. Create a systemd init script: Add the following to `/etc/systemd/system/sprymesh.service` and run the following command

        [Unit]
        Description=Sprymesh phoenix app
        After=network.target

        [Service]
        User=sprymesh
        WorkingDirectory=/opt/www/sprymesh
        EnvironmentFile=/opt/www/sprymesh/env
        ExecStart=/opt/www/sprymesh/_build/prod/rel/sprymesh/bin/sprymesh foreground
        ExecStop=/opt/www/sprymesh/_build/prod/rel/sprymesh/bin/sprymesh stop
        Restart=always
        StandardInput=null
        StandardOutput=syslog
        StandardError=syslog
        SyslogIdentifier=%n
        KillMode=process
        TimeoutStopSec=5

 9. Reload the systemd scripts. Run the following commands:

        sudo systemctl daemon-reload
        sudo systemctl enable sprymesh.service

 9. Push our code to the server. Push to prod

        git push prod master
        git push prod --tags

 9. Add a `config/prod.secret.exs` on the server, tweak it to include all your info. In my setup, I am passing the db info via `DATABASE_URL` env variable.

        use Mix.Config

        config :danny_web, Danny.Repo,
          adapter: Ecto.Adapters.Postgres,
          pool_size: 10

 11. Add the following to your `/etc/sudoers` file. You need this so that the app user `sprymesh` can restart its' own service

        User_Alias      SPRYMESH_APP = sprymesh
        Cmnd_Alias      SM_PROCESS_CTL = /bin/systemctl daemon-reload, /bin/systemctl start sprymesh.service, /bin/systemctl stop sprymesh.service, /bin/systemctl restart sprymesh.service
        SPRYMESH_APP  ALL = NOPASSWD: SM_PROCESS_CTL

 10. Finally login to the server and run the following commands, to do the first deploy

        sudo su sprymesh
        ~/deploy/after_push

The source for this is hosted at https://github.com/12startupsin12months/blog.12startupsin12months.in/blob/master/_posts/2017-06-27-setting-up-a-simple-deploy-process-using-distillery-and-git-deploy.md
If you find an improvement, send a PR :)
