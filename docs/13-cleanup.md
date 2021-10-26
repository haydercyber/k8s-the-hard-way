# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.
Now that we have finished smoke testing the cluster, it is a good idea to clean up the objects that we created for testing. In this lesson, we will spend a moment removing the objects that were created in our cluster in order to perform the smoke testing
```
# kubectl delete secret kubernetes-the-hard-way 
# kubectl delete svc nginx 
# kubectl delete deployment nginx
```