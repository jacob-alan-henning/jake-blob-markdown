# When libraries are nice to work with and my blogs updated image pipeline

I recently ran into an issue with the blog. Some of the images I were using loaded extremely slowly. 
Which is a big red flag for me because I have 0 traffic going to this website. I was just dumping iphone images 
onto my server; size be dammed. The bottleneck was 100% the size of these images.

I just want to dump pictures from my phone into the blog without thinking about it and maybe it would be nice if the high quality images still existed in source.

Of course they needed to be compressed but where should I actually store them. S3 was the obvious answer; It also has the nice effect of removing the largest bottleneck to amazon. Interstingly during testing s3 was faster on average but the local server could be faster but on average slower. Worth the tradeoff imo. 

I changed my markdown pipeline to compress my pictures and upload them to s3. 
```
- name: optimize image 
        run: |
          find images/ -name "*.jpg" -exec convert {} -auto-orient -resize 1920x1080 -quality 85 -strip {} \;
          find images/ -name "*.jpeg" -exec convert {} -auto-orient -resize 1920x1080 -quality 85 -strip {} \;
          find images/ -name "*.png" -exec convert {} -resize 1920x1080 -quality 95 -strip {} \;

      - name: sync images to S3
        run: |
          aws s3 sync images/ s3://jakeblog-blog-image-cache/ --delete
```

This worked well but I had another problem. I had to manually specify in the markdown to point to the s3 bucket. 
Wouldn't it be nice if the markdown could render into html that used the bucket link by default. 
I am using a library called blackfriday which does a really good job but I figured I would probally have to write a custom parser. 
A cursory google search did not indicate that this was something I could easily modify or change. Looking back I was not looking hard enough and was too lazy to look through the source.

Halfway through building a broken parser. I decided that it was crazy that were wasn't a way to do this with blackfriday without forking it; something I did not want to do. 

I then had crazy idea to ask the llm claude if blackfriday could be extended to change custom output. Not suprisingly it said yes; I have gotten used to llm's being less than truthful.
I demanded some repo's with examples of this being done and to my suprise I found two repositorys implementing the same blackfriday interface where they modyfying the rendering html output. Looking through these code and examples I finally went to the blackfriday source. Now that I had an idea of what to look for lo and behold there was an easy way to change the html. After hacking around I eventually got it to work! 

This is what I ended up with 
```
type jakeRenderer struct {
	cacheUrl string
	*bf.HTMLRenderer
}

func (r *jakeRenderer) RenderNode(w io.Writer, node *bf.Node, entering bool) bf.WalkStatus {
	if node.Type == bf.Image && entering {
		originalDest := string(node.Destination)
		if !strings.HasPrefix(originalDest, "https://") {
			fn := strings.Split(originalDest, "/")
			node.Destination = []byte(r.cacheUrl + fn[1])
		}
	}
	return r.HTMLRenderer.RenderNode(w, node, entering)
}
```
At first glance its a little confusing but its actually quite nice the way the internals are exposed. Your customrenderer is not a layer on top of something but 
the actual rendering logic. By filtering on the node.Type property I can change internal properties for one type of node on blackfridays ast. Then I can modify the fields which change the actual html being rendered. I could have even written my html out using the writer object. Whats nice about this is I can specify that 
what I want changed without having to copy or rewrite any of the original logic. 

I think this is great because I have access to all the fields the default renderer does and I can defer back to it for tokens I want to leave as is. 

This actually reminds me of a youtube video I watched the other day about library design. The idea was that a good library is not opinionated but simply solves a problem for you and exposes all of the fields. I think that is correct. I did not create another layer to apply on top of the renderer I was able to get into the guts and selectively make the changes I needed. 

This was a really pleasant suprise; it was my own fault for not figuring it out sooner and needing to ask an llm. However this may be the ideal use case for llm's. Really good search; because I was able to discover and understand the right way to use a library. This solved my problem in an elegant way where I would have have to spent hours or days trawling google and github. The llm made it seem less costly to dig into source and find examples of the interface. I often struggle with 
how much time to sink into library code. Often it can be a dead end and a considerable time sink to understand if an approach is viable. 

For me typing has never been a real bottleneck; researching and understanding is the bottleneck. So the llm giving me examples allowed me to look through the source and implementations in a few hours. From their I could make a change I understood; Now that is a real time saver. I'm sure my answer exists on google but not easily in the format I expected. 

Now the images on my blog works like this. When they get commited to source nothing happens until the pipeline runs. Then it compresses and uploads them to an s3 bucket. When the blog transforms the markdown into html all of the articles reference the s3 bucket behind a feature flag and the fullsize images are still accessible to users. 

This was a very pleasant experience. I made it easier to write posts and serve them.
