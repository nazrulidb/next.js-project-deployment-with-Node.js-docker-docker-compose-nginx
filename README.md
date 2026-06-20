# next.js-project-deployment-with-Node.js-docker-docker-compose-nginx

1. Create a non-root user:
   
          adduser ubuntu
3. add the user to sudo group:
   
       vi /etc/sudoers
       add the line
        username ALL=(ALL) NOPASSWD:ALL
