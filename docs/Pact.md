##

https://martinfowler.com/articles/practical-test-pyramid.html
* The consuming team writes automated tests with all consumer expectations
* They publish the tests for the providing team
* The providing team runs the CDC tests continuously and keeps them green
* Both teams talk to each other once the CDC tests break



## Consumer Test (our team)
Our microservice consumes the weather API. So it's our responsibility to write a consumer test that defines our expectations for the contract (the API) between our microservice and the weather service.

First we include a library for writing pact consumer tests in our build.gradle:
```
testCompile('au.com.dius:pact-jvm-consumer-junit_2.11:3.5.5')
```
Thanks to this library we can implement a consumer test and use pact's mock services:
```
@RunWith(SpringRunner.class)
@SpringBootTest
public class WeatherClientConsumerTest {

    @Autowired
    private WeatherClient weatherClient;

    @Rule
    public PactProviderRuleMk2 weatherProvider =
            new PactProviderRuleMk2("weather_provider", "localhost", 8089, this);

    @Pact(consumer="test_consumer")
    public RequestResponsePact createPact(PactDslWithProvider builder) throws IOException {
        return builder
                .given("weather forecast data")
                .uponReceiving("a request for a weather request for Hamburg")
                    .path("/some-test-api-key/53.5511,9.9937")
                    .method("GET")
                .willRespondWith()
                    .status(200)
                    .body(FileLoader.read("classpath:weatherApiResponse.json"),
                            ContentType.APPLICATION_JSON)
                .toPact();
    }

    @Test
    @PactVerification("weather_provider")
    public void shouldFetchWeatherInformation() throws Exception {
        Optional<WeatherResponse> weatherResponse = weatherClient.fetchWeather();
        assertThat(weatherResponse.isPresent(), is(true));
        assertThat(weatherResponse.get().getSummary(), is("Rain"));
    }
}
```
* If you look closely, you'll see that the `WeatherClientConsumerTest` is very similar to the `WeatherClientIntegrationTest`. 
* **Instead of using Wiremock for the server stub we use Pact this time.** 
* In fact the **consumer test works exactly as the integration test, we replace the real third-party server with a stub, define the expected response and check that our client can parse the response correctly**. In this sense the WeatherClientConsumerTest is a narrow integration test itself. 
* **The advantage over the wiremock-based test is that this test generates a pact file** (found in `target/pacts/&pact-name>.json`) each time it runs. 
* **This pact file describes our expectations for the contract in a special JSON format**. 
* **This pact file can then be used to verify that our stub server behaves like the real server**. 
* We can take the pact file and hand it to the team providing the interface. 
* They take this pact file and write a provider test using the expectations defined in there. This way they test if their API fulfils all our expectations.

You see that this is where the consumer-driven part of CDC comes from. The **consumer drives the implementation of the interface by describing their expectations**. The provider has to make sure that they fulfil all expectations and they're done. No gold-plating, no YAGNI and stuff.

Getting the pact file to the providing team can happen in multiple ways. A simple one is to check them into version control and tell the provider team to always fetch the latest version of the pact file. A more advances one is to use an artifact repository, a service like Amazon's S3 or the pact broker. Start simple and grow as you need.

In your real-world application you don't need both, an integration test and a consumer test for a client class. The sample codebase contains both to show you how to use either one. If you want to write CDC tests using pact I recommend sticking to the latter. The effort of writing the tests is the same. Using pact has the benefit that you automatically get a pact file with the expectations to the contract that other teams can use to easily implement their provider tests. Of course this only makes sense if you can convince the other team to use pact as well. If this doesn't work, using the integration test and Wiremock combination is a decent plan b.

### Provider Test (the other team)
The provider test has to be implemented by the people providing the weather API. We're consuming a public API provided by darksky.net. In theory the darksky team would implement the provider test on their end to check that they're not breaking the contract between their application and our service.

Obviously they don't care about our meager sample application and won't implement a CDC test for us. That's the big difference between a public-facing API and an organisation adopting microservices. **Public-facing APIs can't consider every single consumer out there or they'd become unable to move forward. Within your own organisation, you can — and should**. Your app will most likely serve a handful, maybe a couple dozen of consumers max. You'll be fine writing provider tests for these interfaces in order to keep a stable system.

The providing team gets the pact file and runs it against their providing service. 
* To do so they **implement a provider test that reads the pact file**,
* stubs out some test data and 
* runs the expectations defined in the pact file against their service.

The pact folks have written several libraries for implementing provider tests. Their main GitHub repo gives you a nice overview which consumer and which provider libraries are available. Pick the one that best matches your tech stack.

For simplicity let's assume that the darksky API is implemented in Spring Boot as well. In this case they could use the Spring pact provider which hooks nicely into Spring's MockMVC mechanisms. A hypothetical provider test that the darksky.net team would implement could look like this:
```
@RunWith(RestPactRunner.class)
@Provider("weather_provider") // same as the "provider_name" in our clientConsumerTest
@PactFolder("target/pacts") // tells pact where to load the pact files from
public class WeatherProviderTest {
    @InjectMocks
    private ForecastController forecastController = new ForecastController();

    @Mock
    private ForecastService forecastService;

    @TestTarget
    public final MockMvcTarget target = new MockMvcTarget();

    @Before
    public void before() {
        initMocks(this);
        target.setControllers(forecastController);
    }

    @State("weather forecast data") // same as the "given()" in our clientConsumerTest
    public void weatherForecastData() {
        when(forecastService.fetchForecastFor(any(String.class), any(String.class)))
                .thenReturn(weatherForecast("Rain"));
    }
}
```
You see that all the provider test has to do is to load a pact file (e.g. by using the `@PactFolder` annotation to load previously downloaded pact files) and then define how test data for pre-defined states should be provided (e.g. using Mockito mocks). There's no custom test to be implemented. These are all derived from the pact file. It's important that the provider test has matching counterparts to the provider name and state declared in the consumer test.

### Provider Test (our team)
We've seen how to test the contract between our service and the weather provider. With this interface our service acts as consumer, the weather service acts as provider. Thinking a little further we'll see that our service also acts as a provider for others: We provide a REST API that offers a couple of endpoints ready to be consumed by others.

As we've just learned that contract tests are all the rage, we of course write a contract test for this contract as well. Luckily we're using consumer-driven contracts so there's all the consuming teams sending us their Pacts that we can use to implement our provider tests for our REST API.

Let's first add the Pact provider library for Spring to our project:
```
testCompile('au.com.dius:pact-jvm-provider-spring_2.12:3.5.5')
```
Implementing the provider test follows the same pattern as described before. For the sake of simplicity I simply checked the pact file from our simple consumer into our service's repository. This makes it easier for our purpose, in a real-life scenario you're probably going to use a more sophisticated mechanism to distribute your pact files.
```
@RunWith(RestPactRunner.class)
@Provider("person_provider")// same as in the "provider_name" part in our pact file
@PactFolder("target/pacts") // tells pact where to load the pact files from
public class ExampleProviderTest {

    @Mock
    private PersonRepository personRepository;

    @Mock
    private WeatherClient weatherClient;

    private ExampleController exampleController;

    @TestTarget
    public final MockMvcTarget target = new MockMvcTarget();

    @Before
    public void before() {
        initMocks(this);
        exampleController = new ExampleController(personRepository, weatherClient);
        target.setControllers(exampleController);
    }

    @State("person data") // same as the "given()" part in our consumer test
    public void personData() {
        Person peterPan = new Person("Peter", "Pan");
        when(personRepository.findByLastName("Pan")).thenReturn(Optional.of
                (peterPan));
    }
}
```
The shown `ExampleProviderTest` needs to provide state according to the pact file we're given, that's it. Once we run the provider test, Pact will pick up the pact file and fire HTTP request against our service that then responds according to the state we've set up.

