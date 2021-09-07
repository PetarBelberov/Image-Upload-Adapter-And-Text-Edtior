# Image Upload Adapter And Text-Edtior in CKEditor 5 (CodeIgniter 4 is used)

## Developed in PHP, and CodeIgniter 4, the controller upload image method follows the concept that you could adapt it to any programming language and framework

#### The custom image upload adapter is an upload architecture in CKEditor 5. There are a couple of things that you should follow if you want the upload functionality to run successfully. The first thing is to set the URL in the constructor that points to the controller's upload image method:

#### ``` this.url = base_url + '/Admin/BlogController/uploadImage'; ```

#### The second thing is the ```_sendRequest``` in the Upload Adapter which prepares the data and sends the request. The line ```data.append( 'blog_text', file );``` must point to the input or text area name ```name="blog_text"``` in the view - html. The same name must be placed in the controller's upload image method - ```$avatar = $this->request->getFile('blog_text');```

#### The third thing to watch out for is in the ClassicEditor function in the Upload Adapter, the line: ```document.querySelector( '#editor' )```. The id should be placed again in the view where is the input or text area. For example: ```<textarea name="blog_text" id="editor"...```


## Controller Upload Method
```
public function uploadImage() {

		$validate = $this->validate([
			'blog_text' => [
				'uploaded[blog_text]',
				'mime_in[blog_text,image/jpg,image/jpeg,image/png,image/gif]',
				'max_size[blog_text,4096]',
			]
		]);
		
		$data = [];
        if ($validate) {
			$avatar = $this->request->getFile('blog_text');
			$file_name = $avatar->getName();
			$path = getcwd();

			// Checks whether the image exists
			if (!file_exists($path . '/public/assets/images/blog/' . $file_name)) {
				$image = \Config\Services::image()
				->withFile($avatar)
				->resize(740, 455, true, 'width')
				->save($path . '/public/assets/images/blog/' . $file_name, 80);
			}
			
			$data = [
				'upload' =>  true,
				'url'  => base_url('public/assets/images/blog/' . $file_name)
			];
        }
		else {
			$data = [
				'upload' =>  false,
        	];
			return redirect()->to('/admin/blog'); 
		}
		return $this->response->setJSON($data);
    }
```

## Upload Adapter
```
class CKUploadImageAdapter {
    constructor( loader ) {
        // The file loader instance to use during the upload.
        this.loader = loader;
        this.url = base_url + '/Admin/BlogController/uploadImage';
    }

     // Starts the upload process.
     upload() {
        return this.loader.file
            .then( file => new Promise( ( resolve, reject ) => {
                this._initRequest();
                this._initListeners( resolve, reject, file );
                this._sendRequest( file );
            } ) );
    }

    // Aborts the upload process.
    abort() {
        if ( this.xhr ) {
            this.xhr.abort();
        }
    }

    // Initializes the XMLHttpRequest object using the URL passed to the constructor.
    _initRequest() {
        const xhr = this.xhr = new XMLHttpRequest();

        // Note that your request may look different. It is up to you and your editor
        // integration to choose the right communication channel. This example uses
        // a POST request with JSON as a data structure but your configuration
        // could be different.
        xhr.open( 'POST', this.url, true );
        
        xhr.responseType = 'json';
    }

    // Initializes XMLHttpRequest listeners.
    _initListeners( resolve, reject, file ) {
        const xhr = this.xhr;
        const loader = this.loader;
        const genericErrorText = `Couldn't upload file: ${ file.name }.`;

        xhr.addEventListener( 'error', () => reject( genericErrorText ) );
        xhr.addEventListener( 'abort', () => reject() );
        xhr.addEventListener( 'load', () => {
            const response = xhr.response;

            // This example assumes the XHR server's "response" object will come with
            // an "error" which has its own "message" that can be passed to reject()
            // in the upload promise.
            //
            // Your integration may handle upload errors in a different way so make sure
            // it is done properly. The reject() function must be called when the upload fails.
            if ( !response || response.error ) {
                return reject( response && response.error ? response.error.message : genericErrorText );
            }

            // If the upload is successful, resolve the upload promise with an object containing
            // at least the "default" URL, pointing to the image on the server.
            // This URL will be used to display the image in the content. Learn more in the
            // UploadAdapter#upload documentation.
            resolve( {
                default: response.url
            } );
        } );

        // Upload progress when it is supported. The file loader has the #uploadTotal and #uploaded
        // properties which are used e.g. to display the upload progress bar in the editor
        // user interface.
        if ( xhr.upload ) {
            xhr.upload.addEventListener( 'progress', evt => {
                if ( evt.lengthComputable ) {
                    loader.uploadTotal = evt.total;
                    loader.uploaded = evt.loaded;
                }
            } );
        }
    }

    // Prepares the data and sends the request.
    _sendRequest( file ) {
        // Prepare the form data.
        const data = new FormData();

        data.append( 'blog_text', file );

        // Important note: This is the right place to implement security mechanisms
        // like authentication and CSRF protection. For instance, you can use
        // XMLHttpRequest.setRequestHeader() to set the request headers containing
        // the CSRF token generated earlier by your application.
        this.xhr.setRequestHeader('x-csrf-token', 'csrf_test_name');
        // Send the request.
        this.xhr.send( data );
    }
}

function CKUploadImageAdapterPlugin( editor ) {
    editor.plugins.get( 'FileRepository' ).createUploadAdapter = ( loader ) => {
        return new CKUploadImageAdapter( loader );
    };
}

ClassicEditor
    .create( document.querySelector( '#editor' ), {
        extraPlugins: [ CKUploadImageAdapterPlugin ],
    } )
    .then( editor => {
        return editor.getData();
    } )
    .catch( error => {}, );
```

## The View - HTML
```
<?php echo form_open_multipart('admin/blog/update', ['class' => 'edit-blog']); ?> 
    <div class="form-group">
        <textarea name="blog_text" id="editor" type="text" class="form-control" placeholder="<?= lang('Blog Text') ?>"><?php echo $article['blog_text']; ?></textarea>
    </div>
    <button id="getData" class="btn btn-primary shadow-2 mb-4"><?= lang('Edit') ?></button>
<?php echo form_close(); ?>
```
### References
https://ckeditor.com/docs/ckeditor5/latest/framework/guides/deep-dive/upload-adapter.html
https://stackoverflow.com/questions/46765197/how-to-enable-image-upload-support-in-ckeditor-5

