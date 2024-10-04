- В pci.yml добавил роль mobi_pci_bookworm.

- В /mobi_pci_bookworm/vars/main.yml заменил libpam-cracklib на libpam-pwquality (в Debian 12 заменили пакет).

- В /mobi_pci_bookworm/tasks/main.yml заменил устаревшую директиву include на include_tasks.

- В /mobi_pci_bookworm/tasks/p6.yml в 1 и 2 тасках заменил pam_tally.so на pam_faillock.so.

- В /mobi_pci_bookworm/tasks/p6.yml в 3 таске заменил pam_cracklib.so на pam_pwquality.so.

- В /mobi_pci_bookworm/tasks/p13.yml подкорректировал регулярное выражение, теперь будут заменяться не только раскомментированные строки, но и закомментированные. Так же убрал директиву `backrefs`, в данном контексте, она мешала создавать строки, если они вообще отсутствуют в файле. Т.е. принцип работы директивы на данный момент такой: она проходит по всем переменным из `items`; если в файле есть закомментированная строка, то она будет расскомантированна и установленно необходимое значение; если есть незакомментированная строка с другим значением, она будет заменена на строку с необходимым значением; есть строка отстсвует, она будет создана в конце.