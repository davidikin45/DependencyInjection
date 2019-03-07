# .NET Core Dependency Injection

* [Microsoft Documentation](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-2.2)
* [Dependency Injection in ASP.NET Core](https://app.pluralsight.com/library/courses/aspdotnet-core-dependency-injection/table-of-contents)
* If there are multiple registrations for the same service type the last registered type wins.
* TryAdd will only register if there is not already a service defined for that service type.
* services.TryAddEnumerable prevents duplication of service type. Useful for composite services which send notifications.
* Resolve multiple services with IEnumerable<IServiceType> or IServiceType[].
* The most parameter rich constructor will be used to create object.

## Safe Registrations
* Singleton Service validation occurs in development to ensure scoped services aren't injected.

![alt text](img/safe.png "CDN")

## Configuration
```
services.Configure<AppSettings>(Configuration.GetSection("AppSettings"));
services.AddTransient(sp => sp.GetService<IOptions<AppSettings>>().Value);

var settings = Configuration.GetSection(sectionKey).Get<AppSettings>();
```

## Notifications
```
servcies.AddSingleton<EmailNotificationService>();
servcies.AddSingleton<SmsNotificationService>();

services.AddSingleton<INotificationService>(sp => new CompositeNotificationService(new INotificationService[]
{ 
	sp.GetRequiredService<EmailNotificationService>(),
	sp.GetRequiredService<SmsNotificationService>()
}));
```

## Open Generic Services
```
services.AddSingleton(typeof(IInterface<>), typeof(Implementation<>));
```

## Extensions for Cleaner Code
```
public static class ConfigurationServiceCollectionExtensions
{
	public static IServiceCollection AddAppConfiguration(this IServiceCollection services)
	{
	}
}
```
## Activator Utilities
* Can be used to construct an object not registered in container with an IServiceProvider.

## Action Injection
* [FromServices] attribute allows dependencies to be resolved for a single action.
 
## Middleware Injection

* Constructor injection only allows singleton service lifetime.
* Can pass dependencies into InvokeAsync which supports all service lifetimes.

```
public class Middleware
{
	private readonly RequestDelegate _next;
	Public Middleware(RequestDelegate next)
	{
		_next = next;
	}
	
	public async Task InvokeAsync(HttpContext context, UserManager<User> userManager)
	{
	
	}
}
```

## View Injection
* @inject AppSettings AppSettings

## Manaully creating scope
```
public class Program
{
	public static async Task Main (string[] args)
	{
		var webHost = CreateWebHostBuilder(args).Build();

		using (var scope = webHost.Services.CreateScope())
		{
			var serviceProvider = scope.ServiceProvider;

			var hostingEnvironment = serviceProvider.GetRequiredService<IHostingEnvironment>();
			var appLifetime = serviceProvider.GetRequiredService<IApplicationLifetime>();
			
			if (hostingEnvironment.IsDevelopment())
			{
				var ctx = serviceProvider.GetRequiredService<TennisBookingDbContext>();
				await ctx.Database.MigrateAsync(appLifetime.ApplicationStopping);

				try
				{
					var userManager = serviceProvider.GetRequiredService<UserManager<TennisBookingsUser>>();
					var roleManager = serviceProvider.GetRequiredService<RoleManager<TennisBookingsRole>>();

					await SeedData.SeedUsersAndRoles(userManager, roleManager);
				}
				catch (Exception ex)
				{
					var logger = serviceProvider.GetRequiredService<ILoggerFactory>().CreateLogger("UserInitialisation");
					logger.LogError(ex, "Failed to seed user data");
				}
			}
		}

		webHost.Run();
	}

	public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
		WebHost.CreateDefaultBuilder(args)
			.ConfigureServices(services => services.AddAutofac())
			.UseStartup<Startup>();
}
```

## Scrutor - Simple .NET Core container assembly scanning and decorators
```
services.Scan(scan => scan.FromAssemblyOf<Interface>()
						  .AddClasses(c => c.AssignableTo<Interface>())
						  .AsImplementedInterfaces()
						  .WithScopedLifetime());
```

```
services.Scan(scan => scan.FromAssemblyOf<Interface>()
						  .AddClasses(c => c.AssignableTo<ScopedInterface>())
						  .As<Interface>()
						  .WithScopedLifetime())
						  .AddClasses(c => c.AssignableTo<SingletonInterface>())
						  .As<Interface>()
						  .WithSingletonLifetime());
```

Decorator functionality is useful for wrapping libraries with caching.
```
public CachedWeatherForecaster : IWeatherForecaster
{
	private readonly IWeatherForecaster _weatherForecaster;
	private readonly IDistributedCache<Result> _cache;
	
	public CachedWeatherForecaster(IWeatherForecaster weatherForecaster, IDistributedCache<Result> cache)
	{
		_weatherForecaster = weatherForecaster;
		_cache = cache;
	}
	
	public async Task<CurrentWeatherResult> GetCurrentWeatherAsync()
	{
		var cacheKey = $"current_weather_{DateTime.UtcNow:yyyy_MM_dd}";

		var (isCached, forecast) = await _cache.TryGetValueAsync(cacheKey);

		if (isCached)
			return forecast;

		var result = await _weatherForecaster.GetCurrentWeatherAsync();

		await _cache.SetAsync(cacheKey, result, 60);

		return result;
	}
}

services.AddHttpClient<IWeatherClient, WeatherApiClient>();
services.AddSingleton<IWeatherForecaster, WeatherForecaster>();
services.Decorate<IWeatherForecaster, CachedWeatherForecaster>();
```

## Autofac
* Autofac.Extensions.DependencyInjection

```
using Autofac;
using Autofac.Extensions.DependencyInjection;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using System;

namespace AspNetCore.Base.DependencyInjection
{
    public static class AutofacExtensions
    {
        public static IServiceCollection AddAutofac(this IServiceCollection services)
        {
            return services.AddSingleton<IServiceProviderFactory<ContainerBuilder>, AutofacServiceProviderFactory>();
        }

        public static IWebHostBuilder UseAutofac(this IWebHostBuilder builder)
        {
            return builder.ConfigureServices(services => services.AddAutofac());
        }

        private class AutofacServiceProviderFactory : IServiceProviderFactory<ContainerBuilder>
        {
            public ContainerBuilder CreateBuilder(IServiceCollection services)
            {
                var containerBuilder = new ContainerBuilder();

                containerBuilder.Populate(services);

                return containerBuilder;
            }

            public IServiceProvider CreateServiceProvider(ContainerBuilder builder)
            {
                var container = builder.Build();
                return new AutofacServiceProvider(container);
            }
        }
    }
}
```
```
public class Program
{
	public static async Task Main (string[] args)
	{
		var webHost = CreateWebHostBuilder(args).Build();

		using (var scope = webHost.Services.CreateScope())
		{
			var serviceProvider = scope.ServiceProvider;

			var hostingEnvironment = serviceProvider.GetRequiredService<IHostingEnvironment>();
			var appLifetime = serviceProvider.GetRequiredService<IApplicationLifetime>();
			
			if (hostingEnvironment.IsDevelopment())
			{
				var ctx = serviceProvider.GetRequiredService<TennisBookingDbContext>();
				await ctx.Database.MigrateAsync(appLifetime.ApplicationStopping);

				try
				{
					var userManager = serviceProvider.GetRequiredService<UserManager<TennisBookingsUser>>();
					var roleManager = serviceProvider.GetRequiredService<RoleManager<TennisBookingsRole>>();

					await SeedData.SeedUsersAndRoles(userManager, roleManager);
				}
				catch (Exception ex)
				{
					var logger = serviceProvider.GetRequiredService<ILoggerFactory>().CreateLogger("UserInitialisation");
					logger.LogError(ex, "Failed to seed user data");
				}
			}
		}

		webHost.Run();
	}

	public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
		WebHost.CreateDefaultBuilder(args)
			.UseAutofac())
			.UseStartup<Startup>();
}
```
```
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using System;
using Autofac;

public class Startup
{
	public Startup(IConfiguration configuration)
	{
		Configuration = configuration;
	}

	public IConfiguration Configuration { get; }

	public void ConfigureServices(IServiceCollection services)
	{            
		services.Configure<CookiePolicyOptions>(options =>
		{                
			options.CheckConsentNeeded = context => true;
			options.MinimumSameSitePolicy = SameSiteMode.None;
		});

		services.AddMvc()
			.AddRazorPagesOptions(options =>
			{
				options.Conventions.AuthorizePage("/FindAvailableCourts");
				options.Conventions.AuthorizePage("/BookCourt");
				options.Conventions.AuthorizePage("/Bookings");
			})
			.SetCompatibilityVersion(CompatibilityVersion.Version_2_2);

		services.AddIdentity<TennisBookingsUser, TennisBookingsRole>()
			.AddEntityFrameworkStores<TennisBookingDbContext>()
			.AddDefaultUI()
			.AddDefaultTokenProviders();

		services.AddDbContext<TennisBookingDbContext>(options =>
			options.UseSqlServer(
				Configuration.GetConnectionString("DefaultConnection")));           
	}
	
	// This method is called after ConfigureServices
	public void ConfigureContainer(ContainerBuilder builder)
	{

	}

	public void Configure(IApplicationBuilder app, IHostingEnvironment env, IServiceProvider serviceProvider)
	{
		if (env.IsDevelopment())
		{
			app.UseDeveloperExceptionPage();
		}
		else
		{
			app.UseExceptionHandler("/Home/Error");
			app.UseHsts();
		}

		app.UseHttpsRedirection();
		app.UseStaticFiles();
		app.UseAuthentication();
		app.UseCookiePolicy();

		app.UseLastRequestTracking(); // only track requests which make it to MVC, i.e. not static files
		app.UseMvc();
	}
}
```