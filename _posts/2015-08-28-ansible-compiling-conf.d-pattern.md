---
layout: post
title:  "Ansible: conf.d compilation pattern"
date:   2015-08-28 12:05:03
categories: devops
---

It's nice to take a complex application config, say for example the routing rules used in logstash or heka, and split them up in to stand-alone files. Use descriptive filenames and you've got an easy to maintain configuration!

The issue is how to install these configuration files on a remote server. Simply copying the files over will work the first time but in the future you might remove a file from your configuration control. Do you script the removal of the remote files? Perhaps you change your copy script to first delete all files in the folder then reinstall?

Neither of these is a good option. Removing all remote files breaks callback systems. You want to restart processes when files are changed and by deleting you would force this restart each time. And keeping track of which files are removed is more error-prone book-keeping.

The best pattern I have found is to compile the smaller config files in to a larger config file. This makes diffing/callbacks easy and free you from book keeping on which files are present on the remote system.

Ansible has a useful module called (`assemble`)[http://docs.ansible.com/ansible/assemble_module.html] which concatenates static files in to one larger file. Which might get you pretty far but I need template evaluation.

Getting an ansible run to evaluate templates and concatenate them is a bit of ansible gymnastics. Here's what you'll need:

    1. A folder full of your broken down configuration templates
    2. A special template in that same folder which can evaluate the templates

Your ansible role will be laid out as such:

```
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
```

It's important to put the number at the beginning of the filename. The glob tool lists the files in lexographical order and this way you'll always have a consistent rendering.

The file `roles/logstash/templates/logstash.conf.j2.noglob` is the root template which iterates on the variable:

```
{% for template_name in lookup('fileglob', '../roles/logstash-elasticsearch/templates/*.j2', wantlist=True) %}
  ### START PARTIAL {{ template_name }} ###
  {{ lookup('template', template_name) }}
  ### END PARTIAL ###
{% endfor %}
```

The tricks here are `lookup('fileglob', $PATTERN, wantlist=True)` and `lookup('template', ...)`.

`fileglob`: much like a shell script glob pattern, this lets you expand a path with wildcards. My biggest confusion was how to keep the code general, what path was the code being run from?

Turns out "it depends". When `lookup('fileglob')` is run from a roles varfile it's relative to the roles files directory. When run inline from a roles template file it's relative to the playbook's folder. I put my playbooks in "ROOT/playbooks" so that's why I have to do "../roles". You might just need to do "roles/".

`template`: evaluate a template at that point. Luckily the `fileglob` module provided a full path so there's no confusion about where to find the template. I use the `| basename` filter in the template so I can strip off my workstation's path but still know what file generate the content.
