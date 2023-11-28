---
layout: post
title:  "How to use Mailjet templates with Symfony mailer"
date:   2023-11-22 23:14:02 +0100
categories: jekyll update
---
First you will obviously need to install symfony mailer using bash <code>composer require symfony/mailer</code>

You will also need mailjet bridge to do this make a <code>composer require symfony/mailjet-mailer</code>

Now you must create an interface that extends the symfony <code>Symfony\Component\Mailer\MailerInterface</code>

```php
<?php

//MailjetInterface.php

declare(strict_types=1);

namespace App\Shared\Mailer;

interface MailjetInterface extends MailerInterface
{
}
```

and create the service that implements it

```php
<?php

//Mailjet.php

declare(strict_types=1);

namespace App\Shared\Mailer;

use Psr\EventDispatcher\EventDispatcherInterface;
use Symfony\Component\DependencyInjection\ParameterBag\ParameterBagInterface;
use Symfony\Component\Mailer\Envelope;
use Symfony\Component\Mailer\Event\MessageEvent;
use Symfony\Component\Mailer\Exception\TransportExceptionInterface;
use Symfony\Component\Mailer\Messenger\SendEmailMessage;
use Symfony\Component\Mailer\Transport;
use Symfony\Component\Mailer\Transport\TransportInterface;
use Symfony\Component\Messenger\Exception\HandlerFailedException;
use Symfony\Component\Messenger\MessageBusInterface;
use Symfony\Component\Mime\RawMessage;

final class Mailjet implements EmailInterface
{
    private ?MessageBusInterface $bus = null;
    private ?EventDispatcherInterface $dispatcher = null;

    public function __construct(
        private ParameterBagInterface $parameterBag,
        private TransportInterface $transport,

        MessageBusInterface $bus = null,
        EventDispatcherInterface $dispatcher = null
    ) {
        $this->bus = $bus;
        $this->dispatcher = $dispatcher;
				$this->transport = Transport::fromDsn($this->parameterBag->get('mailer_dsn'));
    }

    public function send(RawMessage $message, Envelope $envelope = null): void
    {
        if (null === $this->bus) {
            $this->transport->send($message, $envelope);

            return;
        }

        $stamps = [];
        if (null !== $this->dispatcher) {
            $clonedMessage = clone $message;
            $clonedEnvelope = null !== $envelope ? clone $envelope : Envelope::create($clonedMessage);
            $event = new MessageEvent($clonedMessage, $clonedEnvelope, (string) $this->transport, true);
            $this->dispatcher->dispatch($event);
            $stamps = $event->getStamps();

            if ($event->isRejected()) {
                return;
            }
        }

        try {
            $this->bus->dispatch(new SendEmailMessage($message, $envelope), $stamps);
        } catch (HandlerFailedException $e) {
            foreach ($e->getNestedExceptions() as $nested) {
                if ($nested instanceof TransportExceptionInterface) {
                    throw $nested;
                }
            }
            throw $e;
        }
    }
}
```

Now, you'll need to create three classes that extends <code>Symfony\Component\Mime\Header\AbstractHeader</code> class

```php
<?php

//TemplateIdHeader.php

declare(strict_types=1);

namespace App\Shared\Mailer\Symfony\Mime\Mailjet\Header;

use Symfony\Component\Mime\Header\AbstractHeader;

final class TemplateIdHeader extends AbstractHeader
{
    private int $templateId;

    public function __construct(string $name, int $templateId)
    {
        parent::__construct($name);

        $this->templateId = $templateId;
    }

    public function getTemplateId(): int
    {
        return $this->templateId;
    }

    public function setTemplateId(int $templateId): self
    {
        $this->templateId = $templateId;

        return $this;
    }

    public function setBody(mixed $body): void
    {
        $this->setTemplateId($body);
    }

    public function getBody(): int
    {
        return $this->getTemplateId();
    }

    public function getBodyAsString(): string
    {
        return (string) $this->getBody();
    }
}
```

```php
<?php

//TemplateLanguageHeader.php

declare(strict_types=1);

namespace App\Shared\Mailer\Symfony\Mime\Mailjet\Header;

use Symfony\Component\Mime\Header\AbstractHeader;

final class TemplateLanguageHeader extends AbstractHeader
{
    private bool $templateLanguage;

    public function __construct(string $name, bool $templateLanguage)
    {
        parent::__construct($name);

        $this->templateLanguage = $templateLanguage;
    }

    public function getTemplateLanguage(): bool
    {
        return $this->templateLanguage;
    }

    public function setTemplateLanguage(bool $templateLanguage): self
    {
        $this->templateLanguage = $templateLanguage;

        return $this;
    }

    public function setBody(mixed $body): void
    {
        $this->setTemplateLanguage($body);
    }

    public function getBody(): bool
    {
        return $this->getTemplateLanguage();
    }

    public function getBodyAsString(): string
    {
        return (string) $this->getBody();
    }
}
```

```php
<?php

//TemplateVariableHeader.php

declare(strict_types=1);

namespace App\Shared\Mailer\Symfony\Mime\Mailjet\Header;

use Symfony\Component\Mime\Header\AbstractHeader;

final class TemplateVariableHeader extends AbstractHeader
{
    private array $templateVariable;

    public function __construct(string $name, array $templateVariable)
    {
        parent::__construct($name);

        $this->templateVariable = $templateVariable;
    }

    public function getTemplateVariable(): array
    {
        return $this->templateVariable;
    }

    public function setTemplateVariable(array $templateVariable): self
    {
        $this->templateVariable = $templateVariable;

        return $this;
    }

    public function setBody(mixed $body): void
    {
        $this->setTemplateVariable($body);
    }

    public function getBody(): array
    {
        return $this->getTemplateVariable();
    }

    public function getBodyAsString(): string
    {
        return json_encode($this->getBody());
    }
}
```

And that’s it. 🥳

## Usage 

```php
<?php

declare(strict_types=1);

namespace App\Email;

final readonly class Foo
{
		public function __construct(private MailjetInterface $mailjet)
		{
		}

		public function __invoke(): void
		{
				$email = (new MailjetEmail())
            ->from('foo@bar.com')
            ->to('john@doe.com')
            ->subject('This is an email')
            ->templateId(100) // Should be your mailjet template id
            ->templateLanguage(true)
            ->templateVariables([
			          'var1' => 'test',
								'var2' => 'test 2'
              ])
            ])
        ;
	
	      $this->mailjet->send($email);
		}
}
```
[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
