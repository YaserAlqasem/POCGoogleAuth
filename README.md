# POCGoogleAuth

This POC showing the process of Sign in with Google and how to read the token and user claims.

1. First check the appsettings.json file. You will see 

"Authentication": {
    "Google": {
      "ClientId": "907678086353-9ml4s2h21nck8e3o5p184c2jck3dnrfe.apps.googleusercontent.com",
      "ClientSecret": "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"
    }
    
 I got these information by creating web appication on Google.
 
 
 2. Check the Startup file. Under the ConfigurationService method you will be able to see Cookies and Google configs, 
 
 services.AddAuthentication(option =>
            {
                option.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
                option.DefaultChallengeScheme = GoogleDefaults.AuthenticationScheme;
            })
                .AddCookie()
                .AddGoogle(GoogleDefaults.AuthenticationScheme, options =>
                {
                    options.ClientId = Configuration["Authentication:Google:ClientId"];
                    options.ClientSecret = Configuration["Authentication:Google:ClientSecret"];
                    options.Scope.Add("profile");
                    options.Scope.Add("email");
                    options.Events.OnCreatingTicket = async context =>
                    {
                        //idToken is the JWT token
                        var idToken = context.TokenResponse.Response.RootElement.GetString("id_token");
                        var user = await VerifyGoogleToken(idToken);
                        var claims = new List<Claim>
                        {
                            new Claim(ClaimTypes.NameIdentifier, user.Id),
                            new Claim(ClaimTypes.Name, user.Name),
                            new Claim(ClaimTypes.Email, user.Email),
                            // add other claims as needed
                        };
                        
                        var identity = new ClaimsIdentity(claims, context.Identity.AuthenticationType);
                        context.Principal.AddIdentity(identity);
                    };
                });
 
 
 At line 33, you can see that I'm calling a method, for now I implement it in the startup class, but it can be somewhere else as a static method.
 
 I used this method to get user info
 
 private async Task<User> VerifyGoogleToken(string idToken)
        {
            using var httpClient = new HttpClient();
            // this google api help us to get the user info using the token.
            var response = await httpClient.GetAsync($"https://oauth2.googleapis.com/tokeninfo?id_token={idToken}");
            if (response.IsSuccessStatusCode)
            {
                var content = await response.Content.ReadAsStringAsync();
                var json = JObject.Parse(content);
                return new User
                {
                    Id = json.Value<string>("sub"),
                    Name = json.Value<string>("name"),
                    Email = json.Value<string>("email")
                };
            }
            else
            {
                throw new Exception($"Failed to verify Google token: {response.ReasonPhrase}");
            }
        }
    }
    
    
3. Check the HomeController, there you will be able to see
 
     public async Task Login()
        {
            // this code is another way to get the information, we can talk about it later
            //await HttpContext.ChallengeAsync(GoogleDefaults.AuthenticationScheme, new AuthenticationProperties()
            //{
            //    RedirectUri = Url.Action("GoogleResponse")
            //});
            
            // Here I'm using the ChallengeAsync method to Authenticate user then Redirect to Home Page.
            await HttpContext.ChallengeAsync(GoogleDefaults.AuthenticationScheme, new AuthenticationProperties()
            {
                RedirectUri = "/"
            });
        }

    // This method also another way to read user info, we can talk about later
    public async Task<IActionResult> GoogleResponse()
        {
            var result = await HttpContext.AuthenticateAsync(CookieAuthenticationDefaults.AuthenticationScheme);

            var claims = result.Principal.Identities.FirstOrDefault().Claims.Select(claim => new
            {
                claim.Issuer,
                claim.OriginalIssuer,
                claim.Type,
                claim.Value
            });

            return Json(claims);
        }
        
        // Here we are removeing all cookies by using the SignOutAsync()
        public async Task<IActionResult> Logout()
        {
            await HttpContext.SignOutAsync();
            return RedirectToAction("Index");
        }
        
        
 Notes: 
 At the Startup class under the ConfigureService method you will see
 
 //services.AddAuthentication(option =>
            //{
            //    option.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
            //    option.DefaultChallengeScheme = GoogleDefaults.AuthenticationScheme;
            //})
            //    .AddCookie()
            //    .AddGoogle(GoogleDefaults.AuthenticationScheme, options =>
            //    {
            //        options.ClientId = Configuration["Authentication:Google:ClientId"];
            //        options.ClientSecret = Configuration["Authentication:Google:ClientSecret"];
            //        options.ClaimActions.MapJsonKey("urn:google:picture", "picture", "url");
            //    });
            
            
    This is another way to configure the thing, we can talk about later
 
