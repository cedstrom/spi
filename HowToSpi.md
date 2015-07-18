# Introduction #
Using an SPI is a great way to create easily extensible programs, this document explains how it can be done.

Our goal is to create a little program that creates thumbnails from files. Since the world of file-types changes constantly it is practically impossible to know which kind of files to support. Therefore we have chosen to use a Service Provider Interface to allow people to easily add support for their own kind of file types.

Note: Most of the example code here is condensed and certain vital things (like catching exceptions) have been left out to increase clarity. You can get the full code at the [InstallExample](InstallExample.md) page

# The Service Provider Interface #
The first thing to do is to define a Service Provider Interface with all the methods required for the service. In this case the most important method will be the RenderedImage render(File) method. However, before the service can simply throw a file to the service provider it has to know if that service provider supports that file. A boolean accepts(File) method will take care of that. Last but not least we'll add a String getDescription() method to be able to get a nice description of the different renderers should we so desire.

Our complete service provider interface:
```
public interface ThumbnailRenderer {
	String getDescription();
	boolean accepts(File file);
	RenderedImage render(File file) throws IOException;
}
```

# Main (The Service) #
Now we need our main program. For that purpose we create a class Thumbnailer. This class will contain the main method and will perform the service (Rendering thumbnails).

We've decided to allow the user to supply files to create thumbnails of as arguments so our simple main method will come down to something like this:
```
	public static void main(String... args) throws Exception {
		for (String name : args) {
			renderThumbNail(name);
		}
	}
```

Each file provided will be rendered in the renderThumbNail method, which will look something like this:
```
	private static void renderThumbNail(String name) {
		File file = new File(name);
		ThumbnailRenderer renderer = findRenderer(file);
		RenderedImage thumbnail = renderer.render(file);
		writeImage(name, thumbnail);
	}
```

The writeImage method writes the image generated by the service provider to a file and is not relevant to our example so it will not be treated in detail. The real magic happens in the findRenderer(File) method which will be covered in the Finding Service Providers section.

# Finding Service Providers #
In principle loading all available service providers is done fairly easily by calling ServiceLoader.load(SpiClass) which returns a ServiceLoader which implements Iterable<SpiClass> so you can just iterate over the result to get all your available service providers.

Now this would load all service providers that correctly implement the service provider interface, but as soon as one load would go wrong it would break the program. To prevent this the ServiceLoader class has been designed to lazily instantiate the service providers, that is at the call of the iterators next() method. This way it is possible to catch the exceptions thrown by each individual loading of a service provider and the code can not be broken by new packages later added to the system.

```
	private static ThumbnailRenderer findRenderer(File file) {
		Iterator<ThumbnailRenderer> renderers = ServiceLoader.load(ThumbnailRenderer.class).iterator();
		while (renderers.hasNext()) {
			try {
				ThumbnailRenderer renderer = renderers.next();
				if (renderer.accepts(file)) {
					return renderer;
				}
			}
			catch (ServiceConfigurationError e){
				// For now just ignore the exceptions
			}
		}
		return null;
	}
```

# The Service Provider #
In a completely different package, no matter where as long as this package includes the service and the service provider interface in its classpath service providers can be created.

Using the @ProviderFor annotation the only thing that is truly necessary is to implement the service provider interface and annotate the class with the @ProviderFor(SpiClass) annotation

```
@ProviderFor(ThumbnailRenderer.class)
public class ImgThumbnailRenderer implements ThumbnailRenderer
```

// TODO Create static class that can be easily used to load service providers for a specific service (The general variant of ThumbnailRenderer#findRenderer(File))