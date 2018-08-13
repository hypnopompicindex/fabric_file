# fabric_file

# -*- coding: UTF-8 -*-
from fabric.api import env, run, prompt, local, get, sudo
from fabric.colors import red, green
from fabric.state import output

env.environment = ""
env.full = False

output['running'] = False

PRODUCTION_HOST = "myproject.com"
PRODUCTION_USER = "myproject"


def dev():
    """ chooses development environment """
    env.environment = "dev"
    env.hosts = [PRODUCTION_HOST]
    env.user = PRODUCTION_USER
    print("LOCAL DEVELOPMENT ENVIRONMENT\n")


def staging():
    """ chooses testing environment """
    env.environment = "staging"
    env.hosts = ["staging.myproject.com"]
    env.user = "myproject"
    print("STAGING WEBSITE\n")


def production():
    """ chooses production environment """
    env.environment = "production"
    env.hosts = [PRODUCTION_HOST]
    env.user = PRODUCTION_USER
    print("PRODUCTION WEBSITE\n")


def full():
    """ all commands should be executed without questioning """
    env.full = True


def deploy():
    """ updates the chosen environment """
    if not env.environment:
        while env.environment not in ("dev", "staging", "production"):
            env.environment = prompt(red('Please specify target environment ("dev", "staging", or "production"): '))
            print

    globals()["_update_%s" % env.environment]()


def _update_dev():
    """ updates development environment """
    run("")  # password request
    print

    if env.full or "y" == prompt(red('Get latest production database (y/n)?'), default="y"):
        print(green(" * creating production-database dump..."))
        run('cd ~/db_backups/ && ./backup_db.bsh --latest')
        print(green(" * downloading dump..."))
        get("~/db_backups/db_latest.sql", "tmp/db_latest.sql")
        print(green(" * importing the dump locally..."))
        local('python manage.py dbshell < tmp/db_latest.sql && rm tmp/db_latest.sql')
        print
        if env.full or "y" == prompt('Call prepare_dev command (y/n)?', default="y"):
            print(green(" * preparing data for development..."))
            local('python manage.py prepare_dev')
    print

    if env.full or "y" == prompt(red('Download media (y/n)?'), default="y"):
        print(green(" * creating an archive of media..."))
        run('cd ~/project/myproject/media/ '
            '&& tar -cz -f ~/project/myproject/tmp/media.tar.gz *')
        print(green(" * downloading archive..."))
        get("~/project/myproject/tmp/media.tar.gz",
            "tmp/media.tar.gz")
        print(green(" * extracting and removing archive locally..."))
        for host in env.hosts:
            local('cd media/ '
                '&& tar -xzf ../tmp/media.tar.gz '
                '&& rm tmp/media.tar.gz')
        print(green(" * removing archive from the server..."))
        run("rm ~/project/myproject/tmp/media.tar.gz")
    print

    if env.full or "y" == prompt(red('Update code (y/n)?'), default="y"):
        print(green(" * updating code..."))
        local('git pull')
    print

    if env.full or "y" == prompt(red('Migrate database schema (y/n)?'), default="y"):
        print(green(" * migrating database schema..."))
        local("python manage.py migrate --no-initial-data")
        local("python manage.py syncdb")
    print
