## Setting up Windows WSL for the experiment

Dec/11/2020. Contributed by Andrew Jackman (aj6eb). 

(Felix: These are verified to work. Some steps may be overkill though. TODO: validate & simplify. )

* Install WSL  see https://docs.microsoft.com/en-us/windows/wsl/install-win10
   	You want to opt in to the **dev builds** for windows insider program to get a build capable of the simplified install if you want to go that route. Taking the more involved manual option doesnt look to bad either 

* Get Ubuntu version from Microsoft store (I chose v20.04 LTS) 

* If you run into a sudo command that will not run because of conflicts with bash then take a look at https://askubuntu.com/questions/103643/cannot-echo-hello-x-txt-even-with-sudo 

* Commands for hard linking (I know we talked about it in class but I couldn't remember which session) https://www.cyberciti.biz/faq/creating-hard-links-with-ln-command/