---
title: Automating Deployment using Jenkins, Ansible and AWS
date: 2023-11-18 12:40:02
tags:
categories: [Blogs]
sticky: 999
---

# Automating Deployment using Jenkins,Ansible and AWS

In today's fast-paced software development environment, the efficient and reliable deployment of applications is crucial. A Continuous Integration/Continuous Deployment (CI/CD) pipeline plays a pivotal role in automating and streamlining the software development lifecycle. This pipeline integrates Jenkins, Ansible, and cloud services from AWS and Azure to deliver a comprehensive solution for your software deployment needs.

In this article, we will demonstrate the following :

- The Web Application will be deployed using python Flask framework.

- The Web Application should be accessible via web browser from anywhere on port 4000.

- EC2's and their security groups should be created on AWS console with Terraform.

- The rest of the process has to be controlled with control node which is connected SSH port.

- PostgreSQL and Flask App should be running in containers

- We are going to use AWS ECR as an image repository, to create Docker Containers with 2 managed nodes (postgresql and Flask instances).

Project GitHub Repo: [https://github.com/Malek-Zaag/Automation-pipeline-with-Ansible-and-AWS](https://github.com/Malek-Zaag/Automation-pipeline-with-Ansible-and-AWS)

### Project Requirements and architecture :

- Virtual machine for all the workload (Azure VM)

- 2 Virtual machines for application (Flask) and postgresql as docker images

- Ansible

- AWS CLI to fetch images from AWS ECR

![](https://cdn-images-1.medium.com/max/2000/0*ZOOHua7PkbsDLKI_)

### Coding The application:

Our application is a simple python Flask application that has a default route, add_book route and a delete route.

```python
 @app.route('/')
    def index():
        books = Book.query.all()
        return render_template('index.html', books=books)

    @app.route('/add_book', methods=['POST','GET'])
    def add_book():
        if request.method == 'POST':
            title = request.form['title']
            author = request.form['author']
            publication_date = request.form['publication_date']
            book = Book(title=title, author=author, publication_date=publication_date)
            db.session.add(book)
            db.session.commit()
            return redirect(url_for('index'))
        elif request.method =='GET':
            return render_template('form.html')


    @app.route('/delete_book/<int:id>')
    def delete_book(id):
        book = Book.query.get(id)
        db.session.delete(book)
        db.session.commit()
        return redirect(url_for('index'))py

```

### Creating custom Postgres docker image:

As we want to have a custom user and password for our database, we opted for a building a new docker image for it and it creates a table called books and has persistent volumes for our data.

```dockerfile
 FROM postgres:12.16-bullseye

 ENV POSTGRES_USER=admin
 ENV POSTGRES_PASSWORD=admin
 ENV POSTGRES_HOST=localhost

 VOLUME [ "/var/lib/postgresql/data" ]
 WORKDIR /
 COPY init.sql /docker-entrypoint-initdb.d/

 EXPOSE 5432
```

Init.sql:

```sql
CREATE TABLE book ( id SERIAL PRIMARY KEY, title VARCHAR(255), author VARCHAR(255), publication_date DATE );
```

After starting our containers, the application is working fine on our local machine :

![](https://cdn-images-1.medium.com/max/2000/0*DKnpE7gOidyhxbCp)

![](https://cdn-images-1.medium.com/max/2000/0*oUqw3qrINeO4rGHw)

### Writing Infrastructure manifest :

As we mentioned earlier, we will be provisioning a jenkins server and it will be also operating as ansible admin server.

![](https://cdn-images-1.medium.com/max/2000/0*SGeQAHsJsPzx41jK)

Finished writing infrastructure manifests :

![](https://cdn-images-1.medium.com/max/2000/0*uoVwcvC9zKadiiXN)

Now it is time to provision the jenkins main server which will execute our pipeline, for that i used also the IAC approach :

terraform apply - auto-approve - var-file=terraform.tfvars

![](https://cdn-images-1.medium.com/max/2000/0*UF1RGBIERhVvqnKM)

Now it is time to setup the jenkins server and install the appropriate plugins :

![](https://cdn-images-1.medium.com/max/3118/0*lzDFk6DCOvpjxWmm)

We need to setup our AWS access tokens in the machine so it can apply the manifest files:

![](https://cdn-images-1.medium.com/max/2104/0*upsECL4EKhjVNJhD)

### Configuring aws ECR :

Amazon Elastic Container Registry (Amazon ECR) is a fully managed container registry offering high-performance hosting, so you can reliably deploy application images and artifacts anywhere.

![](https://cdn-images-1.medium.com/max/2726/0*pU-jwCzR0gBzxQ0z)

We create the private repository :

![](https://cdn-images-1.medium.com/max/2000/0*MRzhKYrGDamOwraH)

### Pushign images to ECR :

To push our local images to the repository that we have previously created, we need to tag the image like the following :

Docker tag local_image_name repo_name:local_image

![](https://cdn-images-1.medium.com/max/2096/0*aesQ0qJtmbjZWD-D)

Database :

![](https://cdn-images-1.medium.com/max/2312/0*IGu9owxZK2pjTVR8)

Now for ansible to connect to machine, we need to copy the private key to the jenkins server :

![](https://cdn-images-1.medium.com/max/2000/0*99ugvog6ioqY7gpI)

One more thing we needed to do is copying manually the vars.yaml file from our machine to the jenkins server because it contains vulnerable credentials and it cannot be pushed to git:

![](https://cdn-images-1.medium.com/max/2000/0*5SVNAcZ2O6-Z9a_O)

After a lot of troubleshooting and tweaking of the ansible playbook, our pipeline executed successfully :

![](https://cdn-images-1.medium.com/max/2000/0*atzFCCJFQ9b8azGY)

Our playbook consists of installing required packages, installing docker and aws cli, and finally connecting docker to our private repository :

![](https://cdn-images-1.medium.com/max/2000/0*L4pWjMZdXpfdV4hR)

We can now access our Flask application located on the first machine, it seems working perfectly :

![](https://cdn-images-1.medium.com/max/2000/0*UXBQKpOMC3Pi9OCo)

![](https://cdn-images-1.medium.com/max/2000/0*EhMZZTx0WJAlVDGQ)

Before closing up, i want to explain a stage in our pipeline :

```groovy

stage("Adding public IPs to /etc/hosts") {
steps {
dir("./Infrastructure/EC2/") {
sh "terraform output > file.txt"
sh "cut -d '=' -f 2 file.txt"
sh "sed 's/\"//g; s/=//g' file.txt > res.txt"
sh 'awk \' { t = $1; $1 = $2; $2 = t; print; } \' res.txt > f.txt'
sh "cat f.txt"
sh 'sudo -- sh -c "cat f.txt >> /etc/hosts"'
}
}
}
```

This little script takes the two ip addresses of the EC2s from the terraform output and it concatenates them into /etc/hosts file as :

```bash
ip_address hostname
```

So ansible can resolve ip-one and ip-two to their correct addresses.
