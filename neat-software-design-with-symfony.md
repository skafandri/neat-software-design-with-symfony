 # The best software architecture

 When designing software, there are many aspects that you should take into consideration.
 Performance (I know, is one of the most overloaded terms), scalability, security, usability..
 Those are desirable software qualities.
 They can be measurable and be part of specifications as **Up to 2000 concurrent connections, the maximum allowed response time is 200ms**.
 Sometimes they are less defined and harder to assess like **Works smoothly on major mobile devices**.
 Achieving or improving one of this areas requires different techniques and patterns which often pull in different directions.
 For example, an online banking system can't afford to have a [Login with Facebook] button to increase it's usability.

 Scalability is a problem that we are trying to solve since forever (assuming that we consider the apparition of computers as the beginning of life).

 The first scalability challenge I encountered in my life was to store a computer program on more than one floppy disc (yes, this was horizontal scalability). Some time later, I had to change many programs to run on 32 bit processors. Couple of years later, again, let's support processors with more than one core.
 Fast forward closer to present time, we need to write programs and/or alter existing ones to run on multiple computers.

 Now I want to ask the question: what software design tips are the most useful to write (scalable, secure, ergonomic.. insert any desirable aspect here) software?

 Here are some statements I made at some point.
 - The best architecture is the one that supports multiple floppy discs.
 - The best architecture is the one that supports color and monochrome displays.
 - The best architecture is a distributed system.

What I have learned from my own advices, is that most of them has become obsolete.
It turns out that the only advice I can give and guarantee won't get outdated soon is:
- The best architecture is the one that is **easy to change**.

Now most of us build software on top of other software. This includes libraries, frameworks, operating systems and so on. A key point to make a program easy to change is to make it as independent as possible from the underlying pieces. In the recent years, software frameworks has become so popular that the question of using one is not valid anymore, the question is rather which one to use (including homemade frameworks). The second question, which is often overlooked is **how?** Usually every framework will provide detailed documentation about how to install it, use it and even couple of *best practices*. Unfortunately, framework authors, sometimes unintentionally, will try to lock you into their framework and tools by showing you the *easy way* to work with their framework.

We will take Symfony framework as an example. Browsing the [best practices](http://symfony.com/doc/current/best_practices/creating-the-project.html) section from the documentation, under the **Structuring the Application** section you can read:
> src/AppBundle/, stores the Symfony specific code (controllers and routes), your domain code (e.g. Doctrine classes) and all your business logic;

Hmm, put all the Symfony code and my business logic in one place? Plus a hint to bind my domain code to an other framework (Doctrine)? No thanks! I'll step over this trap.

Note that I am not trying to devalue the quality of the Symfony documentation. It is perfectly fine to build small applications. I think is acceptable to throw couple of hundreds or thousands of lines of code into that AppBundle. But when things start to scale up and get more complicated, you will have a hard time maintaining it.

If you don't believe it, the line just before the previously mentioned one reads as:
> var/sessions/, stores all the session files generated by the application;

Do you know any decently sized application that an afford to store sessions on disc? So clearly, this part of the documentation would be more suitable under the title "Symfony Best Practices for small applications".

Now that we know how not to design a Symfony application, we will try to follow a step by step tutorial on how to build a professional scale application using Symfony framework.

# Another Jobeet?
About 8 years ago, the [Jobeet tutorial](http://symfony.com/legacy/doc/jobeet/1_2/en/01?orm=Doctrine) was my way into learning Symfony. For nostalgic reasons, I will keep the same theme but using the latest stable Symfony version (3.2 as of the time of writing). We will also put some more complex business rules to make the exercise more interesting.

## Project description
We will develop a simple job posting website with the following features:

1. A user can create many job posts which will be in draft status by default.
2. A user can acquire a number of credit points that are managed by the following rules:
  1. Publishing a job post costs a configurable number of credit points.
  2. Editing a published job's title or description costs a configurable number of credit points.
3. A user can view any published job post.
4. A user can view all his job posts.

## Prepare your dev environment
In order to avoid any conflicting dependencies, we will work on a clean ubuntu virtual machine. I will use VirtualBox and Vagrant to setup my dev environment but you are free to use any tools you are comfortable with. To follow the next commands step by step, make sure you have VirtualBox and Vagrant installed on your computer.

Go to your project directory and run

````
$ vagrant init ubuntu/xenial64; vagrant up
````

Now we can login into the newly created machine.

````
$ vagrant ssh
````

Install PHP and required extensions

````
$ sudo apt-get update
$ sudo apt-get install -y wget git php php-cli php-curl php-xml php-mbstring php-zip
````

Install composer

````
$ php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
$ php composer-setup.php
All settings correct for using Composer
Downloading 1.2.3...

Composer successfully installed to: /home/ubuntu/composer.phar
Use it: php composer.phar

$ sudo mv composer.phar /usr/bin/composer
````

Initialize a new project

````
$ mkdir /vagrant/jobeet
$ cd /vagrant/jobeet
````

By default `/vagrant` directory will be synced with the directory where you run `vagrant up`. So you should be able to open `your project directory/jobeet` in your favorite IDE.

````
$ composer init --name=tutorial/jobeet -n
$ composer install
````

We have a working dev environment now. We can start writing code.

## Domain objects

So far we have installed PHP and created a composer project without any packages. This is an ideal place to start writing our business objects without being influenced by any third party libraries.

Let's start with the Job domain object.

````php
// src/JobPost.php

namespace Jobeet;

class Job
{
    const STATUS_DRAFT = 0;
    const STATUS_PUBLISHED = 1;
    const STATUS_UNPUBLISHED = 2;

    /** @var  string */
    private $title = '';
    /** @var  string */
    private $description = '';
    /** @var  integer */
    private $status = self::STATUS_DRAFT;
    /** @var  \DateTime */
    private $publishDate;
    /** @var  User */
    private $user;

    /**
     * @return string
     */
    public function getTitle(): string
    {
        return $this->title;
    }

    /**
     * @param string $title
     * @return Job
     */
    public function setTitle(string $title): Job
    {
        $this->title = $title;
        return $this;
    }

    /**
     * @return string
     */
    public function getDescription(): string
    {
        return $this->description;
    }

    /**
     * @param string $description
     * @return Job
     */
    public function setDescription(string $description): Job
    {
        $this->description = $description;
        return $this;
    }

    /**
     * @return int
     */
    public function getStatus(): int
    {
        return $this->status;
    }

    /**
     * @param int $status
     * @return Job
     */
    public function setStatus(int $status): Job
    {
        $this->status = $status;
        return $this;
    }

    /**
     * @return \DateTime|null
     */
    public function getPublishDate()
    {
        return $this->publishDate;
    }

    /**
     * @param \DateTime $publishDate
     * @return Job
     */
    public function setPublishDate(\DateTime $publishDate): Job
    {
        $this->publishDate = $publishDate;
        return $this;
    }

    /**
     * @return User
     */
    public function getUser(): User
    {
        return $this->user;
    }

    /**
     * @param User $user
     * @return Job
     */
    public function setUser(User $user): Job
    {
        $this->user = $user;
        return $this;
    }
}
````

Next, the user object

````
// src/User.php

namespace Jobeet;

class User
{
    /** @var  string */
    private $name;
    /** @var  int */
    private $credit = 0;

    /**
     * @return string
     */
    public function getName(): string
    {
        return $this->name;
    }

    /**
     * @param string $name
     * @return User
     */
    public function setName(string $name): User
    {
        $this->name = $name;
        return $this;
    }

    /**
     * @return int
     */
    public function getCredit(): int
    {
        return $this->credit;
    }

    /**
     * @param int $credit
     * @return User
     */
    public function setCredit(int $credit): User
    {
        $this->credit = $credit;
        return $this;
    }
}
````

Our domain objects are just plain PHP classes without any heavy logic in them. We will try to keep them as simple as possible. Our next step is to start wiring our domain objects to establish the required business rules.

## Business rules

The first business rule `A user can create many job posts which will be in draft status by default` seems trivial and already guaranteed by the `Job:status` field default value. However we need a stronger guarantee, think about when the project will scale up in lines of code and number of people working on it. Yes, you guessed it right, we need unit tests that will ensure all the rules are enforced and that we won't introduce regression bugs while adding new features.

We will use the PHPUnit testing framework.

````
$ composer require phpunit/phpunit --dev
````

Edit `composer.json` and add an `autoload` block

````
"autoload": {
  "psr-4": {
    "Jobeet\\": "src/"
  }
}
````

This will instruct composer to autoload any class with `Jobeet` prefixed namespace from the `src` directory.

To make the change active run

````
composer dumpautoload
````

Add a phpunit.xml under the project's root directory to configure the Jobeet test suite

````
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="https://schema.phpunit.de/5.6/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         backupGlobals="false"
         colors="true"
         verbose="true">

    <testsuite name="Jobeet">
        <directory suffix="Test.php">tests</directory>
    </testsuite>
    <filter>
        <whitelist processUncoveredFilesFromWhitelist="true">
            <directory suffix=".php">src</directory>
        </whitelist>
    </filter>
</phpunit>
````

Now we can run the test suite

````
$ ./vendor/phpunit/phpunit/phpunit
PHPUnit 5.7.2 by Sebastian Bergmann and contributors.

Runtime:       PHP 7.0.8-0ubuntu0.16.04.3
Configuration: /vagrant/jobeet/phpunit.xml
Error:         No code coverage driver is available



Time: 133 ms, Memory: 2.00MB

No tests executed!
````

Obviously we didn't write any tests yet. Before that let's create a shortcut for the phpunit command so we won't have to type that long command all the time.

````
$ mkdir bin; ln -s ../vendor/phpunit/phpunit/phpunit bin/phpunit
````

Now we are ready to write our first test case.

````
// tests/JobTest.php

namespace Tests;

use Jobeet\Job;

class JobTest extends \PHPUnit_Framework_TestCase
{
    public function testJobHasDraftStatusByDefault()
    {
        $job = new Job();

        $this->assertEquals(Job::STATUS_DRAFT, $job->getStatus());
    }
}
````

Running the test suite again should succeed

````
$ ./bin/phpunit
PHPUnit 5.7.2 by Sebastian Bergmann and contributors.

Runtime:       PHP 7.0.8-0ubuntu0.16.04.3
Configuration: /vagrant/jobeet/phpunit.xml

.        1 / 1 (100%)

Time: 166 ms, Memory: 4.00MB

OK (1 test, 1 assertion)
````

Did you notice something interesting? We were able to run our code without the need for a web server or a browser or to boot any framework code. Once you get used to this technique you will experience a very short feedback loop between the moment you add new code and seeing it executing. I can't emphasis enough how practical is this.

Do you remember what was the most important aspect of a good design? To be easy to change. I didn't discover yet something that helps to change code better than a good test suite. In fact, given a perfectly designed application without tests you may be afraid to change it and you may introduce bugs when you do. While given a poorly designed application covered with a test suite, you can change it with confidence and improve it's design over time because you know you didn't break anything as long as the tests run *green*.

Let's move on to some more serious logic which is the action of publishing a Job.

````
// src/JobPublisher.php

namespace Jobeet;

class JobPublisher
{
    /** @var  int */
    private $creditsToPublish;

    /**
     * JobPublisher constructor.
     * @param int $creditsToPublish
     */
    public function __construct(int $creditsToPublish)
    {
        if ($creditsToPublish < 1) {
            throw new \InvalidArgumentException('credits to publish must be positive');
        }
        $this->creditsToPublish = $creditsToPublish;
    }


    public function publish(Job $job)
    {
        $user = $job->getUser();
        if ($user->getCredit() < $this->creditsToPublish) {
            throw new \RuntimeException('Not enough credit to publish');
        }

        $user->setCredit($user->getCredit() - $this->creditsToPublish);

        return $job->setStatus(Job::STATUS_PUBLISHED);
    }
}
````

Like we did before, we will write a test case to ensure that the `publish` method works as expected.

````
// test/JobPublisherTest.php

namespace Tests;

use Jobeet\Job;
use Jobeet\JobPublisher;
use Jobeet\User;
use PHPUnit\Framework\TestCase;

class JobPublisherTest extends TestCase
{
    public function testPublishJobAndSubstractUserCredit()
    {
        $job = new Job();
        $user = new User();
        $user->setCredit(10);

        $job->setUser($user);

        $publisher = new JobPublisher(1);
        $publisher->publish($job);

        $this->assertEquals(Job::STATUS_PUBLISHED, $job->getStatus());
        $this->assertEquals(9, $user->getCredit());
    }
}
````

Note that this test case is not enough because it covers only the *happy path*. We should handle all the corner cases as well. So let's add the case of publishing without having enough credit to do so.

````
/**
 * @expectedException \RuntimeException
 * @expectedExceptionMessage Not enough credit to publish
 */
public function testPublishJobWithoutEnoughCredit()
{
    $job = new Job();
    $user = new User();

    $job->setUser($user);

    $publisher = new JobPublisher(1);
    $publisher->publish($job);
}
````

# The sculptor architect

Wait a minute. Shouldn't we design a solution before starting coding?
What happened to UML and all our shiny modeling tools?

You may picture software architecture activity like this

![alt text](./images/architect.png)

Thinking thoroughly about the problem in hand. Modeling objects, interactions, abstractions, dependencies...
Identifying system constraints and balancing compromises.
The [Wikipedia article](https://en.wikipedia.org/wiki/Software_design) provides more details about the subject.

The issue is that a model you create, is merely an **attempt** to reflect the piece of software's architecture.
The only place that holds the **real** design is the code itself.
Moreover, a model is just a communication format (even if you are your only correspondent).

Thus, I rather picture software architecture activity like this

![alt text](./images/sculptor.png)

Progressively shaping the structure and interactions into a meaningful design.
This should be a fun and creative process.

I'll admit, I actually perform the *thinking and drawing* style.
But I think is hard to get good enough at it without mastering the sculptor style first.

Doesn't this sounds like skipping design and just start programming?
Not really, when you program, you just write code that *works*.
When you design, you will often go back and forth with the code.
Constantly reviewing and refining your (and your colleagues) design decisions.
Some people may consider this an apart activity and call it *refactoring*.
I feel it like separating pushing pedals and shifting gears when driving a car with manual transmission. For me is just driving.
Sure you can reach your destination using only the first gear.
But you will reach your destination much slower, while producing much more noise and heat.

So let's shift gears and get back to review the `publish` method.

````
public function publish(Job $job)
{
    $user = $job->getUser();

    if ($user->getCredit() < $this->creditsToPublish) {
        throw new \RuntimeException('Not enough credit to publish');
    }

    $user->setCredit($user->getCredit() - $this->creditsToPublish);

    return $job->setStatus(Job::STATUS_PUBLISHED);
}
````

What's wrong with this method? The guilty line is `$user = $job->getUser();`.

Now the `publish` method *knows* where to find the User of a Job.
It also *knows* that a Job can have exactly one user, which is the only one allowed to publish it.
What if we decide to support a corporate account type that allows multiple users to share a credit account?
What if we decide to add the possibility for our customer support to publish a Job on behalf of a User without decreasing his credit?
We can eliminate this coupling by removing that line and adding the User as a second argument to the method.

The result will look like this

````
public function publish(Job $job, User $user)
{    
    if ($user->getCredit() < $this->creditsToPublish) {
        throw new \RuntimeException('Not enough credit to publish');
    }

    $user->setCredit($user->getCredit() - $this->creditsToPublish);

    return $job->setStatus(Job::STATUS_PUBLISHED);
}
````

`Tests\JobPublisherTest` should be updated to reflect the changes as well.

Now a skeptical person wouldn't be reading this yet.
She would be still starring at the re-factored method and thinking about other ways we could improve it.
What if we would like to give, let's say a Christmas gift and let everybody post jobs for free on Christmas eve?
Shall we create a *special* publisher for that day? Shouldn't we re-factor the publish method to accommodate this change?
In one of my first jobs, I was building an accounting hosted application with MySQL and PHP.
Early in the project, my boss came a long and asked me:

> I would like to give our customers 100% flexibility, they should be able to change anything without needing us.
What options do we have to achieve this?

My answer was: Very simple, give them access to the database and teach them PHP.  
He just smiled and walked away.

A big chunk of software we produce does the following: interpret user input, process it, and produce an output.
Right now I am typing with my keyboard and my text editor is interpreting every keystroke,
parsing the markdown syntax and displaying me a nice preview.
But this text editor is not *flexible enough*.
I can't make those code blocks as references to an online git repository.
So the editor would update the code snippets whenever I push to a particular branch.
If I would want this feature so badly, I would need a backend system that generates this content,
doing the necessary work to keep it up to date. Adding flexibility, inevitably adds complexity to a software.
That's why, in the category of user input processing, programming languages (compilers, linkers, VMs...) are one of the most complex programs.
Note that I am referring to the complexity of writing the software, not using it.
And naturally, programming language is about the most flexible tool you can get to process your input.
Mr skeptical probably got enough of starring at the code snippet and joined us here.
Now he is wondering, what kind of flexibility is better?
And how much flexibility should be built in? Here is a snippet of a code I wrote:

````
reader = new BufferedReader(new FileReader(new File("backup/"+i)));
String[] dates = new String[]{"946684800","978307200","1009843200"};
String[] filters = new String[]{"important","Important","for later"};
````    

This was a one time script I wrote to get some files I needed from a broken HDD before throwing it.
This answer the how much question with the typical cold "it depends".
Because indeed it depends, and every application is unique.
I really can't think about any general rule.
The only thing I am certain about is that every time I hit a flexibility barrier,
the next piece of code I write gets a little better.  
So my advice is learn about your domain and practice. Nothing beats experience here.

There is a mistake I noticed beginner programmers make often.
They get stuck in a detail, sometime to the extent of solving the wrong problem, or solving it in very inefficient way.
In order to make the best decisions, you should always keep the *bird view* on the code.
I personally literally push back my chair and take a global view at the whole application frequently.
The position has no physical impact to improve my view of the code.
Is just a trigger I use to switch the architect hat forget for a while about small details.


Wasn't this supposed to be more about Symfony? Yes, we will learn a lot about Symfony.
But we will thoroughly discuss other aspects of building applications.
Design, testability, performance, scalability and every important property of this application.
Because the definition of what a software architecture should include is: The important stuff.

We already wrote a good deal of business logic (did we?). Let's **plug in** Symfony into our application and see what it can offer.

# Plugging Symfony in

## Install Symfony

What is Symfony? A web framework, right? What does that means exactly?
Symfony is simply a collection of useful components or libraries that provides solutions to common problems we are trying to solve in web development.
At the time of writing, Symfony ships with 34 components, you can check the full list at `vendor/symfony/symfony/src/Symfony/Component/`.
Programmers are lazy, especially the good ones.
That's why in addition to those components, Symfony ships also with some *gluing* code that wires those components together.
If you use the Symfony installer, it works out of the box.
Unfortunately, for many programmers this will remain a *black box*. And I keep hearing *non sense* statements like "oh, Symfony3 has many changes, now **the console** is in `bin/console` not in `app/console` anymore!". Really, what about `symfony/my-awesome-console`?
What we will do next it to unwrap the *Symfony box* slowly so we can understand better how it works. I should warn you, it is way easier than you might think it is.

First, Symfony is a PHP library installable with composer, just like any other package.

````
composer require symfony/symfony
````

We are done, we can start testing our first *Symfony application*. We will indeed create a console application in **symfony/my-awesome-console**.

````
<?php

require_once __DIR__ . '/../vendor/autoload.php';

$application = new \Symfony\Component\Console\Application();
$application->add(new \Symfony\Component\Console\Command\Command('test'));
$application->run();
````

Let's run it

````
$ php symfony/my-awesome-console
Console Tool

Usage:
  command [options] [arguments]

Options:
  -h, --help            Display this help message
  -q, --quiet           Do not output any message
  -V, --version         Display this application version
      --ansi            Force ANSI output
      --no-ansi         Disable ANSI output
  -n, --no-interaction  Do not ask any interactive question
  -v|vv|vvv, --verbose  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug

Available commands:
  help  Displays help for a command
  list  Lists commands
  test
````  

Looks familiar? Yes, that's the Symfony console component in action. Let's try to run a command.

````
$ php symfony/my-awesome-console test

  [Symfony\Component\Console\Exception\LogicException]                   
  You must override the execute() method in the concrete command class.  

test
````

The error message is self explanatory.
It means that we need to extend the `Command` class and override the `execute` method.
Since this is a throw away code that we will use just to get a better understanding, let's create the command within the console file.
We will edit **symfony/my-awesome-console** and add our super simple command.

````
<?php

require_once __DIR__ . '/../vendor/autoload.php';

class TestCommand extends \Symfony\Component\Console\Command\Command
{

    protected function execute(
        \Symfony\Component\Console\Input\InputInterface $input,
        \Symfony\Component\Console\Output\OutputInterface $output
    )
    {
        $output->writeln('This is a test command');
        $output->writeln('<info>This is an info message</info>');
        $output->writeln('<error>This is an error message</error>');
    }

}

$application = new \Symfony\Component\Console\Application();
$application->add(new TestCommand('test'));
$application->run();
````

Run the command again. Hmmm, it works.
