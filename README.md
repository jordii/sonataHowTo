#composer required
    composer require sonata-project/admin-bundle
    composer require sonata-project/doctrine-orm-admin-bundle
    composer require sonata-project/core-bundle

#registerBundles
    new Sonata\CoreBundle\SonataCoreBundle(),
    new Sonata\BlockBundle\SonataBlockBundle(),
    new Sonata\jQueryBundle\SonatajQueryBundle(),
    new Knp\Bundle\MenuBundle\KnpMenuBundle(),
    new Sonata\DoctrineORMAdminBundle\SonataDoctrineORMAdminBundle(),
    new Sonata\AdminBundle\SonataAdminBundle()

# app/config/config.yml
    sonata_admin:
        title:      Acme Demo Bundle
        title_logo: bundles/acmedemo/img/fancy_acme_logo.png
        templates:
                layout:  DemoApiBundle:Admin:standard_layout.html.twig //override default layout if you want.

    sonata_block:
        default_contexts: [cms]
        blocks:
            # Enable the SonataAdminBundle block
            sonata.admin.block.admin_list:
                contexts:   [admin]

#install assets & cc
    php app/console assets:install web
    php app/console cache:clear

#update fosrestbundle config.yml
    - {path: '^/admin', priorities: ['html']}

# app/config/routing.yml
    admin:
        resource: '@SonataAdminBundle/Resources/config/routing/sonata_admin.xml'
        prefix: /admin

    _sonata_admin:
        resource: .
        type: sonata_admin
        prefix: /admin

# Acme/DemoBundle/Resources/config/admin.yml
    services:
        sonata.admin.post:
            class: Acme\DemoBundle\Admin\PostAdmin
            tags:
                - { name: sonata.admin, manager_type: orm, group: "Content", label: "Post" }
            arguments:
                - ~
                - Acme\DemoBundle\Entity\Post
                - ~
            calls:
                - [ setTranslationDomain, [AcmeDemoBundle]]
    //Note: "show_in_dashboard: false" in tags
    //load admin.yml in YouBundleExtensions.php            

#create admin class
    <?php
    // src/Acme/DemoBundle/Admin/PostAdmin.php

    namespace Acme\DemoBundle\Admin;

    use Sonata\AdminBundle\Admin\Admin;
    use Sonata\AdminBundle\Datagrid\ListMapper;
    use Sonata\AdminBundle\Datagrid\DatagridMapper;
    use Sonata\AdminBundle\Form\FormMapper;

    class PostAdmin extends Admin
    {
        // Fields to be shown on create/edit forms
        protected function configureFormFields(FormMapper $formMapper)
        {
            $formMapper
                ->add('title', 'text', array('label' => 'Post Title'))
                ->add('author', 'entity', array('class' => 'Acme\DemoBundle\Entity\User'))
                ->add('body') //if no type is specified, SonataAdminBundle tries to guess it
            ;
        }

        // Fields to be shown on filter forms
        protected function configureDatagridFilters(DatagridMapper $datagridMapper)
        {
            $datagridMapper
                ->add('title')
                ->add('author')
            ;
        }

        // Fields to be shown on lists
        protected function configureListFields(ListMapper $listMapper)
        {
            $listMapper
                ->addIdentifier('title')
                ->add('slug')
                ->add('author')
                ->add(
                    '_action',
                    'actions',
                    array(
                        'actions' => array(
                            'edit'  => array()
                        )
                    )
                )
            ;
        }
    }



#ProductAdmin
class ProductAdmin extends Admin
{

    // Fields to be shown on create/edit forms
    protected function configureFormFields(FormMapper $formMapper)
    {
        $formMapper
            ->add('name', 'text', array('label' => 'Nombre'))
            ->add(
                'Photos',
                'sonata_type_collection',
                array(
                    'required'     => false,
                    'by_reference' => false //important!
                ),
                array(
                    'edit'         => 'inline',
                    'inline'       => 'table',
                    'sortable'     => 'position',
                    'allow_delete' => true //important!
                )
            );
    }

    // Fields to be shown on lists
    protected function configureListFields(ListMapper $listMapper)
    {
        $listMapper
            ->addIdentifier('name')
            ->add(
                '_action',
                'actions',
                array(
                    'actions' => array(
                        'edit' => array()
                    )
                )
            );
    }

#class Product
..
..

/**
* @ORM\OneToMany(targetEntity="Demo\ApiBundle\Entity\Photo", mappedBy="Product",
* cascade={"persist"}, //!important
* orphanRemoval=true   //!important
* )
*/
private $Photos;


public function addPhoto($photo)
{
    $photo->setProduct($this);
    $this->getPhotos()->add($photo);
    $photo->upload('');
}

public function removePhoto($photo)
{
    $this->getPhotos()->removeElement($photo);
}


#UPLOAD METHOD IN PHOTO CLASS

    protected $file;

    public function getFile()
    {
        return $this->file;
    }

    public function setFile($file)
    {
        $this->file = $file;
    }

    public function getAbsolutePath()
    {
        return null === $this->src ? null : $this->getUploadRootDir('') . '/' . $this->src;
    }

    public function getWebPath()
    {
        return null === $this->src ? null : $this->getUploadDir() . '/' . $this->src;
    }

    protected function getUploadRootDir($basepath)
    {
        // the absolute directory path where uploaded documents should be saved
        return $basepath . $this->getUploadDir();
    }

    protected function getUploadDir()
    {
        // get rid of the __DIR__ so it doesn't screw when displaying uploaded doc/image in the view.
        return 'uploads';
    }

    public function upload($basepath)
    {
        // the file property can be empty if the field is not required
        if (null === $this->file) {
            return;
        }

        if (null === $basepath) {
            return;
        }

        // we use the original file name here but you should
        // sanitize it at least to avoid any security issues

        // move takes the target directory and then the target filename to move to
        $this->file->move($this->getUploadRootDir($basepath), $this->file->getClientOriginalName());

        // set the path property to the filename where you'ved saved the file
        $this->setSrc($this->file->getClientOriginalName());

        // clean up the file property as you won't need it anymore
        $this->file = null;
    }


#IMAGE PREVIEW
    # Twig Configuration
    twig:
        debug:            "%kernel.debug%"
        strict_variables: "%kernel.debug%"
        exception_controller: 'FOS\RestBundle\Controller\ExceptionController::showAction'
        form:
            resources:
                - 'DemoApiBundle:Form:fields.html.twig'

    #field template
    {% block preview_widget %}
        {% set path = (form.vars.attr.path is defined) ? form.vars.attr.path:'/uploads/' %}
        {{ form_widget(form,{'attr':{'style':'display:none' }}) }}
        {% if value %}
            <img src="{{ path ~ value }}" name="{{ name }}" width="200"/><br/>
            <span style="font-size:11px">{{ value }}</span>
        {% endif %}
    {% endblock preview_widget %}

    #PhotoAdmin class

        use Demo\ApiBundle\Form\Type\previewType;

        // Fields to be shown on create/edit forms
        protected function configureFormFields(FormMapper $formMapper)
        {
            $fileFieldOptions = array(
                'label'    => 'Foto',
                'required' => false,
                'attr' => array('path' => '/uploads/')
            );
            $formMapper
                ->add('src', new previewType(), $fileFieldOptions)
                ->add('file', 'file');
        }

        ...

        #preview type class
        <?php
        namespace Demo\ApiBundle\Form\Type;

        use Symfony\Component\Form\AbstractType;
        use Symfony\Component\OptionsResolver\OptionsResolverInterface;

        class previewType extends AbstractType
        {
            public function setDefaultOptions(OptionsResolverInterface $resolver)
            {
                $resolver->setDefaults(
                    array()
                );
            }

            public function getParent()
            {
                return 'text';
            }

            public function getName()
            {
                return 'preview';
            }
        }

#EXTRAS
tinymce => https://github.com/stfalcon/TinymceBundle
