# Why Cloud ?
  
Many companies have a hard time maintaining their data centres. It's also inconvenient for new startups to spend a huge amount on infrastructures. A data centre would mean buying a whole system with lots of RAM, CPU & other necessary hardware. Then, hiring some expert guys to setup the whole system & to maintain it. Security, electricity, etc. would add on to the expenditure. 

To make things easy, many companies rely on Cloud-Computing. Here, they just have to think about their work & not worry about un-necessary expenditure. Most of the Cloud Providers work on the agreement of **Pay-as-we-go**, which means that startups don't need a huge amount to setup their business.


# How to work on Clouds ?

Almost all clouds provide a nice GUI interface. However, companies don't prefer a GUI coz it can't automate things. For automation, CLI is used coz commands can easily be scheduled and hence, things can be automated.

# The Problem 

The problem here arises that different clouds have different CLI commands. Hence, it poses a problem for Cloud Engineers.

# The Solution

The solution lies in using a single method which can be used for all the clouds. One such tool is Terraform. A Terraform code is similar for all clouds and it also helps in maintaining records of what all has been done.


# The Project 

In this project, I have launched a web server using terraform code. It's same as the previous task but here, I was used an EFS volume instead of EBS Volume.

The task description is :

1. Create Security group which allow the port 80.

2. Launch EC2 instance.

3. In this Ec2 instance use the existing key or provided key and security group which we have created in step 1.

4. Launch one Volume using the EFS service and attach it in your vpc, then mount that volume into /var/www/html

5. Developer have uploded the code into github repo also the repo has some images.

6. Copy the github repo code into /var/www/html.

7. Create S3 bucket, and copy/deploy the images from github repo into the s3 bucket and change the permission to public readable.

8. Create a Cloudfront using s3 bucket(which contains images) and use the Cloudfront URL to  update in code in /var/www/html




**Step - 1**  First of all, configure your AWS profile in your local system using cmd. Fill your details & press Enter.

                  aws configure --profile Sparsh
                  AWS Access Key ID [****************WO3Z]:
                  AWS Secret Access Key [****************b/hJ]:
                  Default region name [ap-south-1]:
                  Default output format [None]:
                  
                  
                  
**Step - 2** Next, we create a VPC which will be used later.

                  resource "aws_vpc" "sparsh_vpc" {
                    cidr_block = "192.168.0.0/16"
                    instance_tenancy = "default"
                    enable_dns_hostnames = true
                    tags = {
                      Name = "sparsh_vpc"
                    }
                  }
                  
                  
                  
**Step - 3** Now. a subnet needs to be created which will be used to launch the instance later. 

                  resource "aws_subnet" "sparsh_subnet" {
                    vpc_id = "${aws_vpc.sparsh_vpc.id}"
                    cidr_block = "192.168.0.0/24"
                    availability_zone = "ap-south-1a"
                    map_public_ip_on_launch = "true"
                    tags = {
                      Name = "sparsh_subnet"
                    }
                  }


**Step - 4** I have also created a custom security group with all the reqd. permissions. I'll be using this same security group to launch my instance.


                resource "aws_security_group" "sparsh_sg" {

                    name        = "sparsh_sg"
                    vpc_id      = "${aws_vpc.sparsh_vpc.id}"


                    ingress {

                      from_port   = 80
                      to_port     = 80
                      protocol    = "tcp"
                      cidr_blocks = [ "0.0.0.0/0"]

                    }


                    ingress {

                      from_port   = 2049
                      to_port     = 2049
                      protocol    = "tcp"
                      cidr_blocks = [ "0.0.0.0/0"]

                    }



                    ingress {

                      from_port   = 22
                      to_port     = 22
                      protocol    = "tcp"
                      cidr_blocks = [ "0.0.0.0/0"]

                    }




                    egress {

                      from_port   = 0
                      to_port     = 0
                      protocol    = "-1"
                      cidr_blocks = ["0.0.0.0/0"]
                    }


                    tags = {

                      Name = "sparsh_sg"
                    }
                  }


**Step - 5** Now, let's create an EFS Account & configure it.


                  resource "aws_efs_file_system" "sparsh_efs" {
                    creation_token = "sparsh_efs"
                    tags = {
                      Name = "sparsh_efs"
                    }
                  }


                  resource "aws_efs_mount_target" "sparsh_efs_mount" {
                    file_system_id = "${aws_efs_file_system.sparsh_efs.id}"
                    subnet_id = "${aws_subnet.sparsh_subnet.id}"
                    security_groups = [aws_security_group.sparsh_sg.id]
                  }
                  
                  
                  
**Step - 6** Next, we create a Gateway & a Routing Table.

   
                    resource "aws_internet_gateway" "sparsh_gw" {
                    vpc_id = "${aws_vpc.sparsh_vpc.id}"
                    tags = {
                      Name = "sparsh_gw"
                    }
                  }


                  resource "aws_route_table" "sparsh_rt" {
                    vpc_id = "${aws_vpc.sparsh_vpc.id}"

                    route {
                      cidr_block = "0.0.0.0/0"
                      gateway_id = "${aws_internet_gateway.sparsh_gw.id}"
                    }

                    tags = {
                      Name = "sparsh_rt"
                    }
                  }


                  resource "aws_route_table_association" "sparsh_rta" {
                    subnet_id = "${aws_subnet.sparsh_subnet.id}"
                    route_table_id = "${aws_route_table.sparsh_rt.id}"
                  }
                  
                  
**Step - 7** Now, it's time to launch our instance. I've also written a provisioner code to downlaod & setup the Apache Web Server inside this instance.

                          resource "aws_instance" "test_ins" {
                            ami             =  "ami-052c08d70def0ac62"
                            instance_type   =  "t2.micro"
                            key_name        =  "sparsh_key"
                            subnet_id     = "${aws_subnet.sparsh_subnet.id}"
                            security_groups = ["${aws_security_group.sparsh_sg.id}"]


                           connection {
                              type     = "ssh"
                              user     = "ec2-user"
                              private_key = file("C:/Users/AAAA/Downloads/sparsh_key.pem")
                              host     = aws_instance.test_ins.public_ip
                            }

                            provisioner "remote-exec" {
                              inline = [
                                "sudo yum install amazon-efs-utils -y",
                                "sudo yum install httpd  php git -y",
                                "sudo systemctl restart httpd",
                                "sudo systemctl enable httpd",
                                "sudo setenforce 0",
                                "sudo yum -y install nfs-utils"
                              ]
                            }

                            tags = {
                              Name = "my_os"
                            }
                          }
                          
                          
**Step - 8:** Now since our instance is launched, we mount the EFS Volume to /var/www/html folder where all the code is stored. This will ensure no data loss in case the instance crashes or is accidently deleted.

                  resource "null_resource" "mount"  {
                    depends_on = [aws_efs_mount_target.sparsh_efs_mount]
                    connection {
                      type     = "ssh"
                      user     = "ec2-user"
                      private_key = file("C:/Users/AAAA/Downloads/sparsh_key.pem")
                      host     = aws_instance.test_ins.public_ip
                    }
                  provisioner "remote-exec" {
                      inline = [
                        "sudo mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport ${aws_efs_file_system.sparsh_efs.id}.efs.ap-south-1.amazonaws.com:/ /var/www/html",
                        "sudo rm -rf /var/www/html/*",
                        "sudo git clone https://github.com/sparshpnd23/Cloud_task1.git /var/www/html/",
                        "sudo sed -i 's/url/${aws_cloudfront_distribution.my_front.domain_name}/g' /var/www/html/index.html"
                      ]
                    }
                  }


I have also downloaded all the code & images from Github in my local system, so that I can automate the upload of images in s3 later.

        resource "null_resource" "git_copy"  {
          provisioner "local-exec" {
            command = "git clone https://github.com/sparshpnd23/Cloud_task1.git C:/Users/AAAA/Pictures/" 
            }
        }
        
        
        
I have also retrieved the public ip of my instance and stored it in a file locally as it may be used later.

          resource "null_resource" "ip_store"  {
            provisioner "local-exec" {
                command = "echo  ${aws_instance.test_ins.public_ip} > public_ip.txt"
              }
          }

**Step - 9**  Now, we create an S3 bucket on AWS. The code snippet for doing the same is as follows -

          resource "aws_s3_bucket" "sp_bucket" {
            bucket = "sparsh23"
            acl    = "private"

            tags = {
              Name        = "sparsh2301"
            }
          }
           locals {
              s3_origin_id = "myS3Origin"
            }
            
          
**Step - 10**  Now that the S3 bucket has been created, we will upload the images that we had downloaded from Github in our local system in the above step. Here, I have uploaded just one pic. You can upload more if you wish.

            resource "aws_s3_bucket_object" "object" {
              bucket = "${aws_s3_bucket.sp_bucket.id}"
              key    = "test_pic"
              source = "C:/Users/AAAA/Pictures/pic1.jpg"
              acl    = "public-read"
            }
            
    
 **Step - 11** Now, we create a CloudFront & connect it to our S3 bucket. The CloudFront ensures speedy delievery of content using the edge locations from AWS across the world.

           resource "aws_cloudfront_distribution" "my_front" {
             origin {
                   domain_name = "${aws_s3_bucket.sp_bucket.bucket_regional_domain_name}"
                   origin_id   = "${local.s3_origin_id}"

           custom_origin_config {

                   http_port = 80
                   https_port = 80
                   origin_protocol_policy = "match-viewer"
                   origin_ssl_protocols = ["TLSv1", "TLSv1.1", "TLSv1.2"] 
                  }
                }
                   enabled = true

           default_cache_behavior {

                   allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
                   cached_methods   = ["GET", "HEAD"]
                   target_origin_id = "${local.s3_origin_id}"

           forwarded_values {

                 query_string = false

           cookies {
                    forward = "none"
                   }
              }

                    viewer_protocol_policy = "allow-all"
                    min_ttl                = 0
                    default_ttl            = 3600
                    max_ttl                = 86400

          }
            restrictions {
                   geo_restriction {
                     restriction_type = "none"
                    }
               }
           viewer_certificate {
                 cloudfront_default_certificate = true
                 }
          }
          
          
          
Now, we go to /var/www/html & update the link of the images with the link from CloudFront. 

We copy the Cloud Front ID :

![](/images/cf.png)

Now, we go inside the instance & inside our code. We update the code with this Cloud Front Link.

![](/images/Untitled.png)



**Step - 12** Now, we write a terraform code snippet to automatically retrieve the public ip of our instance and open it in chrome. This will land us on the home page of our website that is present in /var/www/html.

            resource "null_resource" "local_exec"  {


            depends_on = [
                null_resource.mount,
              ]

              provisioner "local-exec" {
                  command = "start chrome  ${aws_instance.test_ins.public_ip}"
                     }
            }

**Step - 13** Now, we go to our Command Prompt & write the following command :

              terraform init

After the plugins have been downloaded, we do **terraform apply**.
 ![](/images/tf.png)
 
 We give our approval when the permission is asked :
  ![](/images/tf1.png)
  

Finally, you'll see your home page open up. I had put in just a Hii message & an image. You can write your own code as desired.

![](/images/Untitled1.png)


Viola !! We did it.


Any suggestions are always welcome.


