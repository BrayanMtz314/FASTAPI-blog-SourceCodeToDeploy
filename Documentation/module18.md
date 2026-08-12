# Module 18: Deply To A VPS - Security, NGINX, SSL & Custom Domain

By now we have a complete, functional and tested application, we are ready to deployment!

There are many ways to deploy an application; here, we will examine two options. In this module, we will look at how to deploy the application on a virtual private server (VPS), which is the simplest option as it utilizes a service that facilitates the maintenance of our program. In the final module, we will carry out a container-based cloud deployment, which is the most up-to-date approach.

In this tutorial we'll use linode as our VPS service and Namecheap as our domain name service, so we need to create an account in this services. 

- The services in where we'll deploy our application require health checks of our application, this is because if we don't have a health server the load balancer redirect the traffic to another server. To validate that our server is health we added this endpoint to the main.py file:

```python
## Health Check Endpoint
@app.get("/health")
async def health_check(db: Annotated[AsyncSession, Depends(get_db)]):
    try:
        await db.execute(text("SELECT 1"))
    except Exception as exc:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Database unavailable",
        ) from exc
    return {"status": "healthy"}
```


