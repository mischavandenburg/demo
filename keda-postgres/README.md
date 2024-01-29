# Tutorial: Autoscaling with KEDA on AKS

This tutorial will guide you through setting up a simple auotscaled application based on PostgreSQL queries.

# Creating / Updating the Cluster

Use this command to create an AKS cluster with KEDA enabled:

`az aks create --resource-group myResourceGroup --name myAKSCluster --enable-keda`

Or you can enable it on an existing cluster:

`az aks update --resource-group myResourceGroup --name myAKSCluster --enable-keda`

# Setting up the Database

Install the EDB Operator to allow the creation of databases

`kubectl apply -f https://get.enterprisedb.io/cnp/postgresql-operator-1.22.0.yaml`

Then apply the database.yaml

`kubectl apply -f database.yaml`

After creating the database, let's populate it with some data.

Start by connecting to the pod and running some commands in psql:

```
kubectl exec -it kedapoc-database-1 -- bash
psql
\c app
```

```sql
CREATE SCHEMA IF NOT EXISTS queue;
CREATE TABLE IF NOT EXISTS queue.messages (id SERIAL PRIMARY KEY, status VARCHAR(50));
GRANT USAGE ON SCHEMA queue TO app;
GRANT SELECT ON queue.messages TO app;
```


Now that the tables are created, we can add some data. This will create a simulated message queue with 100 messages in it.

```sql
DO
$$
BEGIN
    FOR counter IN 1..100 LOOP
        INSERT INTO queue.messages (status) VALUES ('new');
    END LOOP;
END
$$;
```

To verify the amount of messages in the queue:

`SELECT count(*) FROM queue.messages WHERE status = 'new';`

This is the number that we'll base the scaling trigger on.

# Deploying Our Application And Scaling It Up

First, we need to get hold of the password for the database:

`kubectl get secret kedapoc-database-app -n kedapoc -o jsonpath='{.data.password}' | base64 --decode`

Modify the deployment.yaml and add the password you retrieved in the step above.

Now we're ready to deploy our application:

`kubectl apply -f deployment.yaml`

With `kubectl get pods -n kedapoc` you can verify that we now have a deployment with 1 replica and a database with 3 replicas running.

```
NAME                              READY   STATUS    RESTARTS   AGE
httpd-frontend-5c57cfb895-blwnn   1/1     Running   0          9s
kedapoc-database-1                1/1     Running   0          4m3s
kedapoc-database-2                1/1     Running   0          2m3s
kedapoc-database-3                1/1     Running   0          69s
```

Now we're ready to apply our scaled object:

`kubectl apply -f keda.yaml`

Check the status of the scaled object with the following command:

`kubectl describe scaledobjects.keda.sh kedapoc-scaledobject -n kedapoc`

# Scaling down

Now let's simulate that our message queue is empty.

To delete all messages:

`DELETE FROM queue.messages;`

It might take some time, but after a while the deployment should scale back down to 1 replica.

# Conclusion

Congratulations! You have successfully scaled your first application with KEDA!

I suggest you dive deeper into the configuration of the ScaledObject here:

https://keda.sh/docs/2.13/concepts/scaling-deployments/

And learn about all the different possible scalers here:

https://keda.sh/docs/2.13/scalers/

If this tutorial was useful to you, you can [buy me a coffee](https://ko-fi.com/mischavandenburg) on my Ko-fi:

https://ko-fi.com/mischavandenburg

I would appreciate it, and I wish you a wonderful day.
