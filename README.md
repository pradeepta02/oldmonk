# OldMonk
OldMonk is an Automation framework for developing tests using Selenium/WebDriver and TestNG

This framework abstracts away the boiler plate code that are needed for setting up any Selenium/WebDriver/TestNG based automation framework. It provides the below abilities:

- Logging
- Generation of screenshots in case of test failure
- Run tests on different browsers/platforms
- Run tests locally, on in-house selenium grid, on sauce labs or on browserstack
- Generate customized reports
- Abstract away driver creation, capability creation based on browser
- Rerun failed test cases
- Inject test data from data source like xml
- Store/retrieve locators from object repositories
- Scalable and Maintainable
- Configurable project parameters like, application url, time waits, selenium grid information etc.

To use the framework, clone the repo and run the command:
```
mvn clean package
```
It will generate `oldmonk-jar-with-dependencies.jar` file. Now create a simple maven project using the command 
```
mvn archetype:generate -DgroupId={project-packaging}
   -DartifactId={project-name}
   -DarchetypeArtifactId=maven-archetype-quickstart
   -DinteractiveMode=false
```

Import this project in eclipse and any other Java IDE. Add the `oldmonk-jar-with-dependencies.jar` file to build path.

### Steps to create a new test script

Create a new file `BaseTest.java` in `src/main/java` folder. We'll be using this as a base test class, that all test script classes will extend. This class will contain few configuration methods, that will initialize the project and create appropriate driver. At the end of the test, it will close the driver.

```
package com.pswain.oldmonk.uitests;

import java.util.Properties;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.remote.DesiredCapabilities;
import org.testng.ITestContext;
import org.testng.annotations.AfterClass;
import org.testng.annotations.BeforeClass;
import org.testng.annotations.BeforeSuite;

import com.pswain.oldmonk.builder.WebCapabilitiesBuilder;
import com.pswain.oldmonk.factory.WebDriverFactory;
import com.pswain.oldmonk.utils.ObjectRepository;
import com.pswain.oldmonk.utils.ProjectConfigurator;

public class BaseTest
{
    protected static Properties config;
    protected WebDriver         driver;

    @BeforeSuite(alwaysRun = true)
    public void beforeSuite(ITestContext context) throws Exception
    {
        config = ProjectConfigurator.initializeProjectConfigurationsFromFile("project.properties");
        ObjectRepository.setRepositoryDirectory(config.getProperty("object.repository.dir"));
    }

    @BeforeClass(alwaysRun = true)
    public void beforeClass(ITestContext context) throws Exception
    {
        final String browser = context.getCurrentXmlTest().getParameter("browser");
        final String version = context.getCurrentXmlTest().getParameter("version");
        final String platform = context.getCurrentXmlTest().getParameter("platform");

        DesiredCapabilities caps = new WebCapabilitiesBuilder().addBrowser(browser)
                .addBrowserDriverExecutablePath(config.getProperty("chrome.driver.path")).addVersion(version)
                .addPlatform(platform).build();
        
        driver = new WebDriverFactory().createDriver(caps);

        driver.get(config.getProperty("url"));
    }

    @AfterClass(alwaysRun = true)
    public void afterClass(ITestContext context) throws Exception
    {
        if (null != driver)
        {
            driver.close();
            driver.quit();
        }
    }
}

```
### Create a project configuration file

Create a new file `project.properties` in the root of the project and add following  lines:

```
# Base application url
url=http://www.yahoo.com

# Directory where element locators will be stored. All element locator files should end with .properties.
# Locators can be specified in this format: locator_name=locator_strategy,locator
# For ex: mail_link=XPATH,//span[text()='Mail']. Supported locator strategies are 'XPATH', 'ID', 'NAME', 
# 'CLASS_NAME', 'TAG_NAME', 'CSS_SELECTOR', 'LINK_TEXT', 'PARTIAL_LINK_TEXT'
object.repository.dir=src/main/resources

# Application modules
application.modules=login,booking,search

# Test Groups
test.groups=smoke,regression

# Number of times to rerun the test in case of test failure. '0' means, failed tests will not be
# executed again
max.retry.count=0

# Path to chromedriver executable file
chrome.driver.path=/Users/pradeepta/Tools/chromedriver
```

### Create Test Script

Create a test script to open browser, navigate to yahoo.com and click on 'mail' link

```
package com.pswain.oldmonk.uitests;

import org.testng.annotations.Test;

import com.pswain.oldmonk.utils.TestDataProvider;

public class TestClass extends BaseTest
{
    @Test(groups = { "smoke" }, dataProvider = "testdataprovider", dataProviderClass = TestDataProvider.class)
    public void testMethod(String email, String password) throws Exception
    {
        HomePage homePage = new HomePage(driver);
        
        MailLoginPage loginPage = homePage.clickOnMailLink();
        loginPage.typeEmailAddress(email);
        loginPage.typePassword(password);
        
        MailHomePage mailHomePage = loginPage.clickOnSigninButton();

        // Do something with mail home page
    }
}
```
Note that, email and password parameters are injected into the test method from outside data source. To create the data source, create a new file in `src/main/resources` directory. For example our testdata.xml file looks like this:

```
<?xml version="1.0" encoding="UTF-8"?>

<testdatasuite>
	<testclass name="com.pswain.oldmonk.uitests.TestClass">
		<dataset>
			<email>test@yahoo.com</email>
			<password>mypassword</password>
		</dataset>
	</testclass>
</testdatasuite>
```
You can have any number of `dataset` sections inside `testclass` section. If there are multiple `dataset` sections, test method will be executed multiple times with different `dataset` test data.

## Create few page classes

```
package com.pswain.oldmonk.uitests;

import java.io.IOException;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.support.PageFactory;

import com.pswain.oldmonk.core.BasePage;
import com.pswain.oldmonk.exception.InvalidLocatorStrategyException;
import com.pswain.oldmonk.exception.PropertyNotFoundException;

public class HomePage extends BasePage
{
    public HomePage(WebDriver driver) throws IOException
    {
        super(driver);
    }

    public MailLoginPage clickOnMailLink() throws PropertyNotFoundException, InvalidLocatorStrategyException
    {
        click("mail_link");
        return PageFactory.initElements(driver, MailLoginPage.class);
    }
}
```

```
package com.pswain.oldmonk.uitests;

import java.io.IOException;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.support.PageFactory;

import com.pswain.oldmonk.core.BasePage;
import com.pswain.oldmonk.exception.InvalidLocatorStrategyException;
import com.pswain.oldmonk.exception.PropertyNotFoundException;

public class MailLoginPage extends BasePage
{
    public MailLoginPage(WebDriver driver) throws IOException
    {
        super(driver);
    }

    public void typeEmailAddress(String emailAddress) throws PropertyNotFoundException, InvalidLocatorStrategyException
    {
        type("email_text_field", emailAddress);
    }

    public void typePassword(String password) throws PropertyNotFoundException, InvalidLocatorStrategyException
    {
        type("passsword_field", password);
    }

    public MailHomePage clickOnSigninButton() throws PropertyNotFoundException, InvalidLocatorStrategyException
    {
        click("signin_button");
        return PageFactory.initElements(driver, MailHomePage.class);
    }
}

```

## Store element locators

To store element locators create .properties files inside src/main/resources folder. Sample locator file will look like this:

```
mail_link=XPATH,//span[text()='Mail']
email_text_field=ID,email
password_field=NAME,password
```

## Run Test Script

To run the test script, we need a testng xml file. Create an xml file in the root of the directory and add the following content:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "http://testng.org/testng-1.0.dtd" >
<suite name="Sample Test Suite" verbose="1" parallel="classes" thread-count="1">
	<listeners>
		<listener class-name="com.pswain.oldmonk.listener.ScreenshotListener" />
		<listener class-name="com.pswain.oldmonk.listener.RetryListener" />
	</listeners>

	<parameter name="browser" value="chrome" />
	<parameter name="version" value="56.0" />
	<parameter name="platform" value="windows 8" />
	<parameter name="testdataxml" value="src/main/resources/testdata.xml" />

	<test name="Sample Smoke Tests">
		<groups>
			<run>
				<include name="smoke" />
			</run>
		</groups>

		<packages>
			<package name="com.pswain.oldmonk.uitests" />
		</packages>

	</test>
</suite>
```
At this point, chrome browser will be launched and test will be executed. Reports will be generated inside test-output folder. Currently supported browsers are `firefox`, `chrome` and `ie`.

# Additional Information

## Creating browser profile

We can set different profile parameters for different browsers using browser profiles. Currently supported browser profiles are ChromeBrowserProfile, FirefoxBrowserProfile and IEBrowserProfile. To use a profile, use it like this:

```
FirefoxBrowserProfile ffProfile = new FirefoxBrowserProfile();
	
ffProfile.setAcceptUntrustedCertificates(true).showDownloadManagerWhenStarting(false)
                .setDownloadDirectory("/Users/pradeepta/Downloads");
        
DesiredCapabilities caps = new WebCapabilitiesBuilder().addBrowser(browser)
                .addBrowserDriverExecutablePath(config.getProperty("gecko.driver.path")).addVersion(version)
                .addPlatform(platform).addBrowserProfile(ffProfile).build();

driver = new WebDriverFactory().createDriver(caps);

```
## Using proxy

To access the websites using proxy, use it like:

```
DesiredCapabilities caps = new WebCapabilitiesBuilder().addBrowser(browser).addProxy("192.168.3.60", 5678).
                .addBrowserDriverExecutablePath(config.getProperty("gecko.driver.path")).addVersion(version)
                .addPlatform(platform).addBrowserProfile(ffProfile).build();
```

## Running tests using selenium GRID

Assuming that selenium Grid is configued on localhost and is running on port 4444, you can run your tests on grid using `GridUrlBuilder`

```
URL remoteHubUrl = new SeleniumGridUrlBuilder().addProtocol(Protocol.HTTP).addSeleniumHubHost("127.0.0.1")
                .addSeleniumHubPort(4444).build();

driver = new WebDriverFactory().createDriver(remoteHubUrl, caps);
```

## Supported element actions

To check the supported element actions/supported selenium commands see [BasePage.java](src/main/java/com/pswain/oldmonk/core/BasePage.java) 

## Failure screenshot

In case of test failure, the framework will generate the failure screenshot and will copy the file to target folder

## Logging

To capture logs, initialize the logger in `@BeforeClass` like this:

```
LoggerUtils.startTestLogging(this.getClass().getSimpleName());
```

This will create a new file in logs folder with the name of test class being executed, and will capture all logs in that file. Ideally it's a good idea to capture the test script logs in separate log files, instead of putting all logs inside a huge log file. It helps in debugging issues faster.

To stop logging, in `@AfterClass` add a line like this:

```
LoggerUtils.stopTestLogging();
```

## Reporting (Alpha)

You can create custom reports by using the [ReporterAPI](src/main/java/com/pswain/oldmonk/core/ReporterAPI.java). Use the api like this:

```
public class HTMLReporter extends TestListenerAdapter implements IReporter
{
    ReporterAPI api;
    PrintWriter pw;

    @Override
    public void generateReport(List<XmlSuite> arg0, List<ISuite> arg1, String arg2)
    {
        try
        {
            api = new ReporterAPI(arg0, arg1, arg2);
        } catch (IOException e)
        {
            e.printStackTrace();
        }

        try
        {
            pw = new PrintWriter(new File("result.html"));
        } catch (FileNotFoundException e)
        {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        
        pw.write("<html><head>Report</head><body>");
        pw.write(api.getParallelRunParameter() + "<br/>" + api.getSuccessPercentageForSuite());
        pw.write(api.getSuiteName() + "<br/>" + api.getTotalFailedMethodsForSuite());
        
        List<String> testNames = api.getAllTestNames();
        for(String test:testNames){
            pw.write("Test Name: " + test + "<br/>");
            ITestNGMethod[] methods = api.getAllMethodsForTest(test);
            for(ITestNGMethod m: methods){
                pw.println(m.getMethodName());
                pw.println(api.getFailedTestScreenshotPath(test, m.getMethodName()));
                pw.println(api.getTestFailureStackTrace(test, m.getMethodName()));
            }
        }
        
        pw.write("</body></html>");
        pw.flush();
        pw.close();
    }
}
```

To generate the report, add the listener to testng listener section

```
<listeners>
	<listener class-name="com.pswain.oldmonk.HTMLReporter" />
</listeners>
```
