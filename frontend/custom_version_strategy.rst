.. index::
   single: Asset; Custom Version Strategy

How to Use a Custom Version Strategy for Assets
===============================================

Asset versioning is a technique that improves the performance of web
applications by adding a version identifier to the URL of the static assets
(CSS, JavaScript, images, etc.) When the content of the asset changes, its
identifier is also modified to force the browser to download it again instead of
reusing the cached asset.

If your application requires advanced versioning, such as generating the
version dynamically based on some external information, you can create your
own version strategy.

.. note::

    Symfony provides various cache busting implementations via the
    :ref:`version <reference-framework-assets-version>`,
    :ref:`version_format <reference-assets-version-format>`, and
    :ref:`json_manifest_path <reference-assets-json-manifest-path>`
    configuration options.

Creating your Own Asset Version Strategy
----------------------------------------

The following example shows how to create a version strategy compatible with
`gulp-buster`_. This tool defines a configuration file called ``busters.json``
which maps each asset file to its content hash:

.. code-block:: json

    {
        "js/script.js": "f9c7afd05729f10f55b689f36bb20172",
        "css/style.css": "91cd067f79a5839536b46c494c4272d8"
    }

Implement VersionStrategyInterface
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Asset version strategies are PHP classes that implement the
:class:`Symfony\\Component\\Asset\\VersionStrategy\\VersionStrategyInterface`.
In this example, the constructor of the class takes as arguments the path to
the manifest file generated by `gulp-buster`_ and the format of the generated
version string::

    // src/Asset/VersionStrategy/GulpBusterVersionStrategy.php
    namespace App\Asset\VersionStrategy;

    use Symfony\Component\Asset\VersionStrategy\VersionStrategyInterface;

    class GulpBusterVersionStrategy implements VersionStrategyInterface
    {
        /**
         * @var string
         */
        private $manifestPath;

        /**
         * @var string
         */
        private $format;

        /**
         * @var string[]
         */
        private $hashes;

        /**
         * @param string      $manifestPath
         * @param string|null $format
         */
        public function __construct(string $manifestPath, string $format = null)
        {
            $this->manifestPath = $manifestPath;
            $this->format = $format ?: '%s?%s';
        }

        public function getVersion(string $path)
        {
            if (!is_array($this->hashes)) {
                $this->hashes = $this->loadManifest();
            }

            return $this->hashes[$path] ?? '';
        }

        public function applyVersion(string $path)
        {
            $version = $this->getVersion($path);

            if ('' === $version) {
                return $path;
            }

            return sprintf($this->format, $path, $version);
        }

        private function loadManifest()
        {
            return json_decode(file_get_contents($this->manifestPath), true);
        }
    }

Register the Strategy Service
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After creating the strategy PHP class, register it as a Symfony service.

.. configuration-block::

    .. code-block:: yaml

        # config/services.yaml
        services:
            App\Asset\VersionStrategy\GulpBusterVersionStrategy:
                arguments:
                    - "%kernel.project_dir%/busters.json"
                    - "%%s?version=%%s"

    .. code-block:: xml

        <!-- config/services.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd"
        >
            <services>
                <service id="App\Asset\VersionStrategy\GulpBusterVersionStrategy">
                    <argument>%kernel.project_dir%/busters.json</argument>
                    <argument>%%s?version=%%s</argument>
                </service>
            </services>
        </container>

    .. code-block:: php

        // config/services.php
        namespace Symfony\Component\DependencyInjection\Loader\Configurator;

        use App\Asset\VersionStrategy\GulpBusterVersionStrategy;
        use Symfony\Component\DependencyInjection\Definition;

        return function(ContainerConfigurator $containerConfigurator) {
            $services = $containerConfigurator->services();

            $services->set(GulpBusterVersionStrategy::class)
                ->args(
                    [
                        '%kernel.project_dir%/busters.json',
                        '%%s?version=%%s',
                    ]
                );
        };


Finally, enable the new asset versioning for all the application assets or just
for some :ref:`asset package <reference-framework-assets-packages>` thanks to
the :ref:`version_strategy <reference-assets-version-strategy>` option:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/framework.yaml
        framework:
            # ...
            assets:
                version_strategy: 'App\Asset\VersionStrategy\GulpBusterVersionStrategy'

    .. code-block:: xml

        <!-- config/packages/framework.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:framework="http://symfony.com/schema/dic/symfony"
            xsi:schemaLocation="http://symfony.com/schema/dic/services https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/symfony https://symfony.com/schema/dic/symfony/symfony-1.0.xsd">

            <framework:config>
                <framework:assets version-strategy="App\Asset\VersionStrategy\GulpBusterVersionStrategy"/>
            </framework:config>
        </container>

    .. code-block:: php

        // config/packages/framework.php
        use App\Asset\VersionStrategy\GulpBusterVersionStrategy;
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $framework) {
            // ...
            $framework->assets()
                ->versionStrategy(GulpBusterVersionStrategy::class)
            ;
        };

.. _`gulp-buster`: https://www.npmjs.com/package/gulp-buster
