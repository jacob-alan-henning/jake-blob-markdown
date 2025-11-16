# A mildly interesting docker default

My blog runs on the smallest vps on aws. My blog is a systemd unit, a bash script, docker compose, and a small Go program. It works, and while I would not pitch it at work, it could serve thousands of requests a minute. Which is thousands of more requests per minute than my blog receives. 

I was taking another look at the deployment and I decided I don't like the idea of bridge networking with the Docker proxy. That helper runs as root only so that it can bind privileged ports and forward packets, it adds another hop even though I only run one container. It seems gross and I could squeeze more out of it. I'm pretty sure that the real bottleneck is bandwidth limit on the vps but during a spike removing the proxy could matter. 

So I moved to host networking. The container process now listens on the host ports. The obvious way is to run it as root, but the application needs to listen on ports 80 and 443.

Linux systems already have a way to let a process bind low ports without making it root. The capability that does this is named
`cap_net_bind_service`, and when the blog binary has it, it can listen on 80 and 443 while running as an ordinary user.

Then I hit a real roadblock. The blogs certificates sit in `/etc/letsencrypt`. Root owns that directory and other users cannot read it. The container mounts it as read only, so a directory that belongs to root combined with a container user that is not root simply fails.

I expected I would have to have some sort of proxy or be stuck with running the container as root.

Then I had a realization; how is the container root able to read the volume bind at all?

Turns out docker doesn't create a new user namespace for containers.

If the container process runs as GID 2200, I can create a real group on the host with GID 2200 and grant that group read access to `/etc/letsencrypt`. Now the container can run as not root and read what it needs.

It feels like that should not work but once the identifiers line up I give the blog binary the capability with a call to `setcap cap_net_bind_service=+ep /jakeblog` it just works.

