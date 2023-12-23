cat << EOF > dns.txt

EOF

awk -F"[][]" '/CS-Production_LAN_HN2/ {gsub(/[{}'\'']/,""); print "hnb" ++count2 " ansible_host=" $2 > "HN2.txt"}' dns.txt
awk -F"[][]" '/CS_Production_LAN_HN1/ {gsub(/[{}'\'']/,""); print "hna" ++count1 " ansible_host=" $2 > "HN1.txt"}' dns.txt

echo "[HN1]" > hosts
cat HN1.txt >> hosts
echo "" >> hosts
echo "[HN2]" >> hosts
cat HN2.txt >> hosts


cat << EOF >> hosts
[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_extra_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
ansible_ssh_private_key_file=${HOME}/.ssh/id_rsa
ansible_user=root
EOF


cat << EOF > playbook.yaml
- hosts: HN1
  become: true
  tasks:
    - name: Set current interface HN1
      set_fact:
        current_interface: "{{ ansible_default_ipv4['interface'] }}"

    - name: Set DNS servers HN1
      shell: resolvectl dns {{ current_interface }} A.A.A.A 8.8.8.8
      become: true
      register: dns_result
      ignore_errors: true

    - name: Check DNS status HN1
      shell: resolvectl status {{ current_interface }}
      become: true

    - name: Handle DNS error HN1
      block:
        - name: Set DNS servers (fallback) HN1
          shell: sudo resolvectl dns {{ current_interface }} ''
          become: true
          when: "'Failed to set DNS configuration: Link' in dns_result.stderr"

        - name: Set DNS servers (fallback) HN1
          shell: sudo resolvectl dns {{ current_interface }} A.A.A.A
          become: true
          when: "'Failed to set DNS configuration: Link' in dns_result.stderr"

- hosts: HN2
  become: true
  tasks:
    - name: Set current interface HN2
      set_fact:
        current_interface: "{{ ansible_default_ipv4['interface'] }}"

    - name: Set DNS servers HN2
      shell: resolvectl dns {{ current_interface }} B.B.B.B 8.8.8.8
      become: true
      register: dns_result
      ignore_errors: true

    - name: Check DNS status HN2
      shell: resolvectl status {{ current_interface }}
      become: true

    - name: Handle DNS error HN2
      block:
        - name: Set DNS servers (fallback) HN2
          shell: sudo resolvectl dns {{ current_interface }} ''
          become: true
          when: "'Failed to set DNS configuration: Link' in dns_result.stderr"

        - name: Set DNS servers (fallback) HN2
          shell: sudo resolvectl dns {{ current_interface }} B.B.B.B
          become: true
          when: "'Failed to set DNS configuration: Link' in dns_result.stderr"
EOF

ansible-playbook -i hosts playbook.yaml
