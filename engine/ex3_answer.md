**uninstall_apache.yml**

```yaml
---
- hosts: web
  name: Uninstall the apache web service
  become: yes
  tasks:
    - name: stop httpd
      service:
        name: httpd
        state: stopped
      ignore_errors: yes

    - name: uninstall apache
      yum:
        name: httpd
        state: absent
```

[演習に戻る](./ex3.html)

