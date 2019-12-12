# A (simple) guide on using Ansible

## Premessa
- **srv2** e' la workstation di 'controllo' da dove useremo ansible
- **srv_remote** e' una macchina FreeBSD 10+ che vogliamo gestire con ansible da srv2
- srv_remote ha sshd in ascolto sulla port 2202
- su srv2 e srv_remote useremo l'utente **dave**
- dave@srv2 ha copiato la sua chiave ssh pubblica su dave@srv_remote


## Installazione
Installiamo python37 su srv2 con:
```
$ sudo pkg install python37
```

Creiamo un virtualenv in $HOME/.venv con:
```
$ python3.7 -m venv .venv
```

Passiamo su bash e usiamo il virtualenv con:
```
$ bash
[dave@srv2 ~]$ source .venv/bin/activate
(.venv) [dave@srv ~]$ 
```

Quindi installiamo ansible con:
```
(.venv) [dave@srv2 ~]$ pip3.7 install ansible
[...]
(.venv) [dave@srv2 ~]$
```

## Inventario
Creiamo una directory che useremo per i file di configurazione di ansible:
```
$ sudo mkdir -p $HOME/ansible
$ cd $HOME/ansible
```

E creiamo il file di inventario:
```
cat > inventory <<EOF
all:
  vars:
    ansible_user: dave
    ansible_become: yes
    ansible_become_method: 'sudo'
  hosts:
    my_hosts:
      ansible_port: 2202
      ansible_host: srv_remote
EOF
```

## Primo test
Assicuriamoci che l'accesso con chiave ssh funzioni dalla workstation dove abbiamo
installato ansible a tutti i server che vogliamo gestire usando ansible e iniziamo
con un semplice comando di prova:
```
(.venv) [dave@srv2 /usr/local/ansible]$ ansible -i inventory -m raw -a 'uptime' my_hosts
my_hosts | CHANGED | rc=0 >>
 8:55PM  up 27 days,  8:35, 1 user, load averages: 0.68, 0.49, 0.44
Shared connection to srv_remote closed.


(.venv) [dave@srv2 /usr/local/ansible]$ 
```

## Installazione pkgs
Bene, adesso vediamo come usare ansible per installare o aggiornare figlet sul srv_remote
usando pkgng (supportato da ansible):
```
(.venv) [dave@srv2 /usr/local/ansible]$ cat > pkgs.yml <<EOF
- hosts: my_hosts

  tasks:
  - name: install/upgrade/confirm figlet package is installed
    become: yes
    become_method: sudo
    pkgng:
      name: figlet
      state: latest
EOF
```

e poi lanciamo:
```
(.venv) [dave@srv2 /usr/local/ansible]$ ansible-playbook -i inventory pkgs.yml

PLAY [my_hosts] *************************************************************************************

TASK [Gathering Facts] ******************************************************************************
ok: [my_hosts]

TASK [install/upgrade/confirm figlet package is installed] ******************************************
changed: [my_hosts]

PLAY RECAP ******************************************************************************************
my_hosts                   : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0
    ignored=0

(.venv) [dave@srv2 /usr/local/ansible]$
```

## Esecuzione comandi
Possiamo anche eseguire comandi con:
```
  tasks:
  - name: Audit install packages
    command: pkg audit -Fqr
    become: yes
    become_user: root
```

Per cui avremo:
```
(.venv) [dave@srv2 /usr/local/ansible]$ ansible-playbook -i inventory pkgs.yml

PLAY [my_hosts] *************************************************************************************

TASK [Gathering Facts] ******************************************************************************
ok: [my_hosts]

TASK [install/upgrade/confirm figlet package is installed] ******************************************
ok: [my_hosts]

TASK [audit install packages] ***********************************************************************
fatal: [my_hosts]: FAILED! => {"changed": true, "cmd": ["pkg", "audit", "-Fqr"], "delta": "0:00:00.47
2263", "end": "2019-12-12 21:07:16.351704", "msg": "non-zero return code", "rc": 1, "start": "2019-12
-12 21:07:15.879441", "stderr": "", "stderr_lines": [], "stdout": "clamav-0.101.4,1\nPackages that de
pend on clamav: \n\nmysql56-server-5.6.45\nPackages that depend on mysql56-server: ", "stdout_lines":
 ["clamav-0.101.4,1", "Packages that depend on clamav: ", "", "mysql56-server-5.6.45", "Packages that
 depend on mysql56-server: "]}

PLAY RECAP ******************************************************************************************
my_hosts                   : ok=2    changed=0    unreachable=0    failed=1    skipped=0    rescued=0
    ignored=0

(.venv) [dave@srv2 /usr/local/ansible]$
```
