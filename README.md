# PaymentMtgoxBundle

This bundle utilizes [MtGox][mtgox]'s IPN (Instant Payment Notification)
providing ability to create for example donation system, or simple shop.
The only thing you are forced to do is to create `Order` class
implementing `OrderInterface` and its doctrine repository (implementing
`OrderRepositoryInterface`) and redirect user to MtGox Payment Page
(implementation of obtaining address to redirect is also included in
this bundle).

Full documentation and big refactor soon, so use only for testing
purposes (I'm using this on production and everything works well, but my
refactor implies your refactor ;) ).

[mtgox]: https://mtgox.com/

## Documentation

### Installation

`PaymentMtgoxBundle` is installed as every symfony bundle (`composer require`, change in `app/AppKernel.php`)

### Configuration

You should first configure [JMSPaymentCoreBundle][jms] (so only put `payments_secret` in `app/config/parameters.yml').
Next get your api key and secret and also put them there. Something like that:

    mtgox_api_key:    your-mtgox-api-key
    mtgox_api_secret: "+your-mtgox-api-secret=="

You should also add IpnController to your `app/config/routing.yml`:

    wikp_payment_mtgox:
        resource: "@WikpPaymentMtgoxBundle/Resources/config/routing.yml"
        prefix:   /mtgox

Next, write your Order entity and its Repository:

```php
<?php

namespace Acme\DemoBundle\Entity;

use Doctrine\ORM\Mapping as ORM;
use JMS\Payment\CoreBundle\Model\PaymentInstructionInterface;
use Wikp\PaymentMtgoxBundle\Plugin\OrderInterface;

/**
 * @ORM\Entity(repositoryClass="Acme\DemoBundle\Entity\OrderRepository")
 * @ORM\Table(name="item_order")
 */
class Order implements OrderInterface
{
    const STATUS_NEW = 0;
    const STATUS_FINISHED = 1;
    const STATUS_CANCELLED = 2;

    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    private $id;

    /**
     * @ORM\OneToOne(targetEntity="JMS\Payment\CoreBundle\Entity\PaymentInstruction")
     * @ORM\JoinColumn(name="payment_instruction_id", referencedColumnName="id")
     */
    private $paymentInstruction;

    /**
     * @ORM\Column(type="smallint")
     */
    private $status;

    public function __construct()
    {
        $this->status = self::STATUS_NEW;
    }

    public function getId()
    {
        return $this->id;
    }

    /**
     * @return \JMS\Payment\CoreBundle\Model\PaymentInstructionInterface
     */
    public function getPaymentInstruction()
    {
        return $this->paymentInstruction;
    }

    public function setPaymentInstruction(PaymentInstructionInterface $paymentInstruction)
    {
        $this->paymentInstruction = $paymentInstruction;
    }

    public function cancel()
    {
        $this->status = self::STATUS_CANCELLED;
    }

    public function approve()
    {
        $this->status = self::STATUS_FINISHED;
    }
}
```

```php
<?php

namespace Acme\DemoBundle\Entity;

use Doctrine\ORM\EntityRepository;
use Wikp\PaymentMtgoxBundle\Plugin\OrderRepositoryInterface;

class OrderRepository extends EntityRepository implements OrderRepositoryInterface
{
    public function getOrderById($id)
    {
        return $this->find($id);
    }
}
```

[jms]: https://github.com/schmittjoh/JMSPaymentCoreBundle

## Obtaining payment URL

You could obtain payment URL where you next should redirect a user in this way:

```php
<?php

namespace Acme\DemoBundle\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\Controller;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Template;

use Wikp\PaymentMtgoxBundle\Plugin\MtgoxPaymentPlugin;
use Wikp\PaymentMtgoxBundle\Mtgox\RequestType\MtgoxTransactionUrlRequest;
use JMS\Payment\CoreBundle\Entity\PaymentInstruction;
use JMS\Payment\CoreBundle\Model\FinancialTransactionInterface;
use Acme\DemoBundle\Entity\Order;


class DemoController extends Controller
{
    /**
     * @Route("/pay/{amount}", name="_demo_pay")
     * @Template()
     */
    public function helloAction($amount)
    {
        // form validation, etc.

        $currency = 'USD'; //change it, store in Order on config, whatever

        /** @var $ppc \JMS\Payment\CoreBundle\PluginController\PluginController */
        $ppc = $this->get('payment.plugin_controller');
        $ppc->addPlugin($this->get('wikp_payment_mtgox.plugin'));

        $ppc->createPaymentInstruction(
            $instruction = new PaymentInstruction(
                $amount,
                $currency,
                MtgoxPaymentPlugin::SYSTEM_NAME
            )
        );

        $em = $this->get('doctrine.orm.entity_manager');
        $order = new Order();
        $order->setPaymentInstruction($instruction);
        $em->persist($order);
        $em->flush();

        if (FinancialTransactionInterface::STATE_PENDING == $instruction->getState()) {
            $urlRequest = new MtgoxTransactionUrlRequest();
            $urlRequest->setAmount($amount);
            $urlRequest->setIpnUrl($this->generateUrl('wikp_payment_mtgox_ipn', array(), true));
            $urlRequest->setDescription( //info for the user visible on the payment page
                sprintf('You are about to pay for order id=%d', $order->getId())
            );
            $urlRequest->setAdditionalData($order->getId()); //could be useful for debugging
            $urlRequest->setCurrency($currency);
            $urlRequest->setReturnSuccess(
                $this->generateUrl('_demo_payment_successful', array(), true)
            );
            $urlRequest->setReturnFailure(
                $this->generateUrl('_demo_payment_canceled', array(), true)
            );

            return $this->redirect(
                $this->get('wikp_payment_mtgox.plugin')->getMtgoxTransactionUrl($urlRequest)
            );
        }
    }

    /**
     * @Route("/pay-success", name="_demo_payment_successful")
     */
    public function paymentSuccessAction()
    {
        return 'User paid for order but his money could not arrive to your wallet already';
    }

    /**
     * @Route("/pay-cancel", name="_demo_payment_canceled")
     */
    public function paymentCancelAction()
    {
        return 'User has clicked Cancel on mtgox.com';
    }
}
```

## IpnController

When you configure bundle and do the payment, `IpnController` should mark it as finished when all the money will be
transfered to your MtGox account.

Remember that you CANNOT debug this on your local machine, because MtGox probably won't be able to access it.
