---
layout: post
title:  "Ansible: conf.d compilation pattern"
date:   2015-08-28 12:05:03
categories: devops
---

Managing complex application configuration
----

It's nice to take a complex application config, say for example the routing rules used in logstash and split them up in to stand-alone files. Use descriptive filenames and you've got an easy to maintain configuration!

How do you install these configuration files on a remote server? Simply copying the files over will work the first time but in the future you might remove a file from your configuration control. Do you script the removal of the remote files? Perhaps you change your copy script to first delete all files in the folder then copy over?

Neither of these is a good option. Removing all remote files breaks callback systems. You want to restart processes when files are changed and by deleting you would force this restart each time. And keeping track of which files are removed is more error-prone book-keeping. Realistically, people just want to add/remove/edit config files in the folder and expect them to show up on the remote server.

To get the intended behavior you should compile the smaller config files in to a larger config file. This makes diffing/callbacks easy and frees you from tracking files.

Get it done with Ansible
------

Ansible has a useful module called [`assemble`](http://docs.ansible.com/ansible/assemble_module.html) which concatenates static files in to one larger file. Which might get you pretty far but I need template evaluation.

Getting an ansible run to evaluate templates and concatenate them will require some gymnastics. Here's what you'll need:

    1. A folder full of your broken down configuration templates
    2. A special template in that same folder which can evaluate the templates

Your ansible role will be laid out as such:

    roles/logstash/
    ├── tasks
    │   └── main.yml
    ├── templates
    │   ├── 16-foo.conf.j2
    │   ├── 35-bar.conf.j2
    │   ├── 40-quuz.conf.j2
    │   └── logstash.conf.j2.noglob
    └── vars
        └── main.yml

It's important to put the number at the beginning of the filename. The glob tool lists the files in lexographical order and this way you'll always have a consistent rendering. And for some applications it matters how things are ordered.

You install the template as always, from `roles/logstash/tasks/main.yml`:

    - name: assemble logstash es config
      template: >
          src=logstash.conf.j2.noglob
          dest=/home/core/logstash-config/logstash.conf

The file `roles/logstash/templates/logstash.conf.j2.noglob` is the root template which iterates on the variable:

    {{ "{% for template_name in lookup('fileglob', '../roles/logstash/templates/*.j2', wantlist=True) "}}%}
        ### START PARTIAL {{ "{{ template_name" }}   }} ###
        {{ "{{ lookup('template', template_name)" }} }}
        ### END PARTIAL ###
    {{ "{% endfor "}}%}

The tricks here are `lookup('fileglob', $GLOB, wantlist=True)`. Getting the parameters correct took some exploration.

`lookup('fileglob)`: much like a shell script glob pattern, this lets you expand a path with wildcards. You must add `wantlist=True` so you can a python list back. My biggets concern with this templating code: how to keep it generalized and independent of location in the project's folder structure.

It's not impossible but there are some got'chas. When `lookup('fileglob')` is run from a roles varfile it's relative to the roles files directory. When run inline from a roles template file it's relative to the playbook's folder. I put my playbooks in `$ROOT/playbooks` so that's why I have to do "../roles". If you're using a standard layout you'll just need `roles/`.

`lookup('template)`: evaluate a template at that point. Luckily the `fileglob` module provided a full path so there's no confusion about where to find the template. I use the `| basename` helper to strip off my workstation's local path but still know what file generated the content.

Done
---

So there ya go, an idempotent, composable config compiler. Neat.