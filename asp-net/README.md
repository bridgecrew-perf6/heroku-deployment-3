# Deploy ASP.NET webapp with PostgreSQL database to Heroku

After following the steps bellow you can have a HTTPS only ASP.NET web application deployed to Heroku at for instance https://my-project-4.herokuapp.com/ without exposing anything from database credentials directly to Github due to provision of source to Github.

### Prerequisities

Install [Docker](https://docs.docker.com/engine/install/) locally.

Register account and/or login to [Heroku](https://id.heroku.com/login)

Download and install the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-command-line) locally.

### New app & pipeline
On [Personal apps](https://dashboard.heroku.com/apps) page create a new project by clicking on **New** &rarr; **Create new app** at the top right.

On **Create New App** page specify your project name url subdomain, i.e. `my-project-4`, in **App name** form field. Choose **Europe** from the **Choose a region** dropdown. Click on the **Add to pipeline...** button.

From **Choose a pipeline** dropdown choose **Create new pipeline**. Specify **Name the pipeline** as i.e. `my-project`. Leave **Choose a stage to add this app to** as `staging`. Afterwards, click the **Create app** button.

Via Terminal login to heroku CLI.
```sh
heroku login
```

Set the heroku app stack to `container`.
```sh
heroku stack:set container --app my-project-4
```

On the project page using the top tab menu bar switch to **Settings** tab and notice the change of **Stack** to **container**;

![4k7kl48g7h](https://user-images.githubusercontent.com/29838606/171693262-9d53eb7a-d5b9-4d04-a3da-569e6fd5f0d9.png)

### PostgreSQL database
On the project page using the top tab menu bar switch to **Resources** tab. Using the **Quickly add add-ons from Elements** add the **Heroku Postgres** add-on. In the newly displayed modal window choose **Hobby Dev - Free** as **Plan name** and bellow click the **Submit Order Form**.

### ASP.NET webapp modifications
In Visual Studio configure project to use postgres db by following the steps bellow.

Preferably using **Package Manager Console** install postgres EF related packages.
```ps
Install-Package Npgsql
Install-Package Npgsql.EntityFrameworkCore.PostgreSQL
```

Edit `MyProject\Program.cs`.
For instance instead of:
```cs
options.UseSqlServer(config.GetConnectionString("Default")));
```
specify:
```cs
options.UseNpgsql(config.GetConnectionString("Default")));
```

For the reason that PostgreSQL is used replace all occurrences of `DateTime.Now` to `DateTime.UtcNow` whereever C# code interacts with database.
For example instead of:
```cs
public DateTime CreatedAt { get; set; } = DateTime.Now;
```
put:
```cs
public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
```

Add `OnModelConfiguring()` method to `MyProject/Database/ApplicationDbContext.cs` as follows in order to specify connection string.
```cs
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
   string name = Environment.GetEnvironmentVariable("DB_NAME")!;
   string host = Environment.GetEnvironmentVariable("DB_HOST")!;
   string port = Environment.GetEnvironmentVariable("DB_PORT")!;
   string username = Environment.GetEnvironmentVariable("DB_USERNAME")!;
   string password = Environment.GetEnvironmentVariable("DB_PASSWORD")!;
   optionsBuilder.UseNpgsql($"database={name};host={host};port={port};username={username};password={password};sslmode=Require;Trust Server Certificate=true;");
}
```

Set connection string also in the main program file `MyProject\Program.cs`. Note that the connection string needs to be dynamic which makes connection string specified in `MyProject\appsettings.json` irrelevant.
```cs
string name = Environment.GetEnvironmentVariable("DB_NAME")!;
string host = Environment.GetEnvironmentVariable("DB_HOST")!;
string port = Environment.GetEnvironmentVariable("DB_PORT")!;
string username = Environment.GetEnvironmentVariable("DB_USERNAME")!;
string password = Environment.GetEnvironmentVariable("DB_PASSWORD")!;
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseNpgsql($"database={name};server={host};port={port};userid={username};password={password};sslmode=Require;Trust Server Certificate=true;"));
```

To setup auto-redirects from HTTP to HTTPS first add `"https_port": 443,` line to `MyProject/appsettings.json` file.
```json
{
  "https_port": 443,
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      ...
```

Then add `using Microsoft.AspNetCore.HttpOverrides;` to the top of `MyProject\Program.cs` file.

Also to `MyProject\Program.cs` file, bellow the line `var app = builder.Build();` add the following.
```cs
app.UseForwardedHeaders(new ForwardedHeadersOptions() {
    ForwardedHeaders = ForwardedHeaders.XForwardedProto
});
app.UseHttpsRedirection();
```

Afterwards at the bottom of the same `MyProject\Program.cs` file replace:
```cs
app.Run();
```
with:
```cs
app.Run("http://*:" + Environment.GetEnvironmentVariable("PORT"));
```

### Create Migrations
Preferably using **Package Manager Console** create **Migrations**.
```ps
Add-Migration InitialCreate
```

### Docker & Heroku build tools
While logged in via heroku CLI run the following command from CLI to retrieve database credentials.
```sh
heroku pg:credentials:url --app my-project-4
```

Note all important database credentials from the output, especially `dbname`, `host`, `port`, `user` and `password`. These are needed to specify connection strings in the ASP.NET app.
```sh
$ heroku pg:credentials:url --app my-project-4
Connection information for default credential.
Connection info string:
   "dbname=d4kt1tre3q9act host=ec2-34-246-227-219.eu-west-1.compute.amazonaws.com port=5432 user=zzakyrrtvowciz password=91c9f031fccdc95f24776a35c0e553c675eac4495483f6e1dce78a54af223507 sslmode=require"
Connection URL:
   postgres://zzakyrrtvowciz:91c9f031fccdc95f24776a35c0e553c675eac4495483f6e1dce78a54af223507@ec2-34-246-227-219.eu-west-1.compute.amazonaws.com:5432/d4kt1tre3q9act
```

Put credentials given above to the following heroku command in order to set environment variables for the project on Heroku.
```sh
heroku config:set --app my-project-4 DB_HOST=ec2-34-246-227-219.eu-west-1.compute.amazonaws.com DB_NAME=d4kt1tre3q9act DB_PASSWORD=91c9f031fccdc95f24776a35c0e553c675eac4495483f6e1dce78a54af223507 DB_PORT=5432 DB_USERNAME=zzakyrrtvowciz
```

Refresh the project page. Using the top tab menu bar switch to **Settings** tab and click on the **Reveal Config Vars** button.

![4tz89i7uuzi](https://user-images.githubusercontent.com/29838606/171695521-b881eb3b-8996-482d-95fc-53c6395400c9.png)

Notice config variables added by the previous command.

![87j879hg7jg](https://user-images.githubusercontent.com/29838606/171696255-a8f14642-176f-49d3-be99-081bc35333cf.png)

Create a new `heroku.yml` file at the root dir, where the `MyProject.sln` file resides, with the following contents.
```yml
build:
  docker:
    web: Dockerfile
```

Create a new `Dockerfile` file at the root dir, where the `MyProject.sln` file resides, with the following contents. Note all occurrences of `MyProject`.
```Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build-env
WORKDIR /app

EXPOSE 80
EXPOSE 443

COPY . .

ARG DB_NAME \
  DB_HOST \
  DB_PORT \
  DB_USERNAME \
  DB_PASSWORD

RUN dotnet restore; \
  dotnet publish ./MyProject/MyProject.csproj -c Release -o out;

RUN echo "$(date) - project name: MyProject"; \
  export PATH="$PATH:/root/.dotnet/tools"; \
  dotnet tool install --global dotnet-ef; \
  echo "$(date) - running migrations"; \
  dotnet ef database update --project ./MyProject; \
  echo "$(date) - migrations completed";

FROM mcr.microsoft.com/dotnet/aspnet:6.0-alpine
WORKDIR /app
COPY --from=build-env /app/out .

CMD ASPNETCORE_URLS=http://*:$PORT dotnet MyProject.dll
```

Push the project named i.e. `MyProject` to Github to it's own new repository.

### Link Heroku project to Github repository
On the project page using the top tab menu bar switch to **Deploy** tab. Click onto **Connect to Github** and hook the Github repository to the Heroku project.

![478gh8kh98](https://user-images.githubusercontent.com/29838606/171699379-b188d202-17bd-48d3-9d49-96c054e29586.png)

### Build docker image locally
As database credentials are involved the docker image build needs to be done locally. Pushing an image to Heroku happens automatically after docker image is built. Pushing to Heroku requires container login.
```sh
heroku container:login
```

Make sure your current directory is set to local github repository. Use the following command with previously gained credentials to build the docker image for the ASP.NET webapp.
```sh
heroku container:push web --app my-project-4 --arg DB_HOST=ec2-34-246-227-219.eu-west-1.compute.amazonaws.com,DB_NAME=d4kt1tre3q9act,DB_PASSWORD=91c9f031fccdc95f24776a35c0e553c675eac4495483f6e1dce78a54af223507,DB_PORT=5432,DB_USERNAME=zzakyrrtvowciz
```
Follow the ouput of the docker image build. It should perform database migrations successfully. In case of an error do not bother attempting to release the image to Heroku.

### Release the image and deploy the webapp to Heroku
Run the following command using heroku CLI and then on the project page using the top tab menu bar switch to **Activity** tab.
```sh
heroku container:release web --app my-project-4
```

![h47g8f9h7gf](https://user-images.githubusercontent.com/29838606/171703704-2069acbb-77ca-4710-907f-3ad1d30fc215.png)

The webapp should be deployed and available to visitors. Click the **Open app** button at the top right which is shown on the picture above.
