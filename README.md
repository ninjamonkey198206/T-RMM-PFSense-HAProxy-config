# T-RMM-PFSense-HAProxy-config

Assumes fully configured public DNS or DDNS, and a functional PFSense installation with existing valid wildcard SSL certificate available. Adjust hostnames, IPs, etc to suit the environment.

##
# HAProxy installation

Go to System > Package Manager

![Screenshot 2022-03-31 125739](https://user-images.githubusercontent.com/24654529/161121728-ae0c7023-9896-4ec4-bb44-03db3760cdb7.png)

Select Available Packages

![Screenshot 2022-03-31 130058](https://user-images.githubusercontent.com/24654529/161121800-e5babfd9-29ed-433a-b0c7-850a5aa5b017.png)

Find and install haproxy-devel

![Screenshot 2022-03-31 130322](https://user-images.githubusercontent.com/24654529/161121985-953e24a6-bcaa-418d-a1e4-1ef62a193623.png)

##
# Firewall configuration

Go to Firewall > Rules

![Screenshot 2022-03-31 135327](https://user-images.githubusercontent.com/24654529/161128877-85aec1f2-c829-4700-81ca-7e78a112d891.png)

Select the WAN tab

![Screenshot 2022-03-31 135621](https://user-images.githubusercontent.com/24654529/161129178-55784d70-87d7-4d1d-b980-80c211b17bd0.png)

Add the following two rules to the bottom of the list, nothing else should be using ports 80 and 443:

![Screenshot 2022-03-31 135726](https://user-images.githubusercontent.com/24654529/161129621-9809859c-f50f-45f9-bef8-635036189fef.png)

HTTP rule:

![Screenshot 2022-03-31 140214](https://user-images.githubusercontent.com/24654529/161130992-8d2af2d4-a448-4b2a-a2dc-f9e01174b85a.png)

Create a duplicate rule, changing the port to 443 and the description for HTTPS

Enable the new rules.

##
# HAProxy backend configuration

Go to Services > HAProxy

![Screenshot 2022-03-31 141001](https://user-images.githubusercontent.com/24654529/161131598-e4caf3a6-fd43-4f35-b0cd-3b1236722a18.png)

Leave the Settings tab alone for now and skip to the Backend tab:

![Screenshot 2022-06-22 122204](https://user-images.githubusercontent.com/24654529/175102215-2a8a59fa-2151-4cf6-9077-47b91077d512.png)

Add backend for rmm. Enter the rmm FQDN (eg, rmm.example.com) into the box titled Name, then click the down arrow in the Server list panel to add a server definition.

Enter the rmm for the server in the Name box, enter the internal IP address of the server, 443 (or other appropriate port) for the Port, and check the Encrypt(SSL) box.

In the Timeout / retry settings section, enter 30000 in Connection timeout and Server timeout.

In the Health checking section, set Health check method to none.

In the Advanced settings section, enter the following in the Backend pass thru box:

```text
http-request add-header X-Forwarded-Host %[req.hdr(Host)]
http-request add-header X-Forwarded-Proto https
```

Scroll down, save, and apply changes when asked.

![rmmhaproxy5](https://user-images.githubusercontent.com/24654529/175108303-64fd386b-135c-42ba-8489-0ef15751cc57.JPG)

Copy the rmm backend. Change the Name entry to match the mesh FQDN (eg, mesh.example.com).

Change the Name for the server to mesh.

In the Timeout / retry settings section, change the entries in Connection timeout and Server timeout to 15000.

In the Advanced settings section, add the following in the Backend pass thru box:

```text
timeout tunnel      15000
```

Scroll down, save, and apply changes when asked.

![rmmhaproxy4](https://user-images.githubusercontent.com/24654529/175109842-1a77fd32-43f3-4399-b10b-8f9320e80ed7.png)

Copy the mesh backend. Change the Name entry to match the mesh FQDN-websockets (eg, mesh.example.com-websockets).

Change the Name for the server to mesh-websockets.

In the Timeout / retry settings section, change the entries in Connection timeout and Server timeout to 3000.

In the Advanced settings section, change the timeout tunnel entry in the Backend pass thru box to 3600000.

Scroll down, save, and apply changes when asked.

##
# Shared HTTP to HTTPS redirect frontend 

Now go to the Frontend tab. Click the button to add a new frontend.

This shared http frontend will redirect all configured entries to their HTTPS equivalent and allow SSL offloading, as well as both internal and external access to the sites/services via URL.

Fill in the entries as shown in the screen capture below:

![Screenshot 2022-03-31 144739](https://user-images.githubusercontent.com/24654529/161137213-1c992c70-c608-48f9-b2ec-6ba3f8852bb1.png)

Scroll to the section titled "Default backend, access control lists and actions" and in the Action Control lists area click the down arrow to create a new acl. Enter rmm in the Name field, change the Expression to Host matches, and enter the FQDN for rmm into the Value field (eg, rmm.example.com).

Copy the rmm acl. Change the Name to api, and the Value field to the FQDN for api into the Value field (eg, api.example.com).

Copy the api acl. Change the Name to mesh, and the Value field to the FQDN for mesh into the Value field (eg, mesh.example.com).

Scroll down to the Actions area of the section and click the down arrow to create a new action. In the Action field, select http-request redirect, enter scheme https into the rule field, and enter rmm into the Condition acl names field.

Copy the action you just created, and change the Condition acl name to api.

Repeat this again, and change the Condition acl name to mesh.

Scroll down and select None for the Default Backend.

![rmmhaproxy2 - Copy](https://user-images.githubusercontent.com/24654529/175117814-e88d87be-99bb-42bc-bcae-2d9e51b9300b.png)

Scroll down to the Advanced settings section. Tick the Use "forwardfor" option box, select http-server-close for Use "httpclose" option, and add/copy-paste
```text
http-request add-header         X-Real-IP %[src]
```
to the Advanced pass thru text box.

![Screenshot 2022-03-31 150416](https://user-images.githubusercontent.com/24654529/161140094-cd0082e0-24b6-4710-817c-6f9a8a59ef75.png)

Save and apply changes. 

# Shared HTTPS frontend

Click the button to add a new frontend.

Fill in the entries as shown in the screen capture below:

![Screenshot 2022-03-31 151854](https://user-images.githubusercontent.com/24654529/161142546-414c9798-8deb-4f0c-bcb3-e2a7d178ca67.png)

No entries are necessary in the Default backend, access control lists and actions section, just make sure to set the Default Backend to None.

As before, scroll down to the Advanced settings section, tick the Use "forwardfor" option box, select http-server-close for Use "httpclose" option, and add/copy-paste
```text
http-request add-header         X-Real-IP %[src]
```
to the Advanced pass thru text box.

Next, scroll down to the SSL Offloading section. In the Certificate area, select the wildcard certificate for the domain and tick the box to Add ACL for certificate Subject Alternative Names.

In the OCSP area, tick the option box.

![Screenshot 2022-03-31 152841](https://user-images.githubusercontent.com/24654529/161150406-a99a9f51-075f-4deb-ac59-28fd803b4b87.png)

Save and apply changes.

This shared HTTPS frontend will provide SSL offloading for ALL HTTPS frontends using it as a shared frontend, while allowing other ACLs and actions to be assigned to individual sub frontends independent from each other, as well as give a visual list of frontends/services that's easier to read than a long list of ACLs.

# Site/service frontend

Click the button to add a new frontend.

Fill out the fields as in the picture below. In the Name field, enter the hostname. In the Description field, enter the FQDN. Set the Status to active, tick the Shared Frontend box, and select https_shared - http as the Primary frontend.

![Screenshot 2022-03-31 161129](https://user-images.githubusercontent.com/24654529/161150507-90dd91b8-6340-49b6-8374-d7f984f10e1a.png)

Scroll to the section titled "Default backend, access control lists and actions" and in the Action Control lists area click the down arrow to create a new acl for your server. Enter the hostname in the Name field, change the Expression to Host matches, and enter the FQDN of the website/service into the Value field. In the Actions area, click the down arrow to create a new action. In the Action field, select Use Backend, select the backend you created earlier, and enter the hostname into the Condition acl names field. Scroll down and select None for the Default Backend.

![Screenshot 2022-03-31 161526](https://user-images.githubusercontent.com/24654529/161152536-2801c030-5101-4e2f-ae7a-e647f7b809c0.png)

Save and apply changes.

The website/service should now be available internally and externally at the configured URL, with SSL encryption, and automatic HTTP to HTTPS forwarding.
