Każdy playbook puszczamu z hasłem, hasło użytkownika szyfrujemy poleceniem:

    ansible-vault create passwd.yml

W pliku zapisujemy hasło w formacie:

    servers_sudo_pass: twoje_hasło

Następnie przy wywołaniu playbook-a dodajemy parametry jak w poniższym przykładzie:

    ansible-playbook -i inventory.yml --ask-vault-pass --extra-vars '@passwd.yml' 01_general-server-configuration.yml

Wtedy ansible poprosi nas o hasło utworzone w pierwszym kroku. Proponuje użyć hasła systemowego (by je pamiętać)