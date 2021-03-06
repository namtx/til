# Install
```ruby
rails active_storage:install
```
`ActiveStorage` easily create 2 tables `active_storage_blobs` and `active_storage_attachments`

- `active_storage_blobs` save about blob file's information such as: `key`, `filename`, `content_type`, `meta_data`, `byte_size`,...

- `active_storage_attachments` is a join table between record and `blobs`, use `polymorphic` technique

# Where to start?
```ruby
rails g scaffold user name:string email: string
```

```ruby
class User < ActiveRecord::Base
  has_one_attached :avatar
end
```

`ActiveStorage` will inject methods for associations between `User` model and `ActiveStorage::Blob` and `ActiveStorage::Attachment` as well as callbacks for deleting files when delete `User`.

Change `User`'s form and controller:

```ruby
# app/views/users/_form.html.erb

<div class="field">
  <%= form.label :avatar %>
  <%= form.file_field :avatar %>
</div>

```
```ruby
# app/controllers/users_controller.rb
def user_params
  params.require(:user).permit(:email, :name, :avatar)
end
```

When submit the form, it will pass `:avatar` as `ActionDispatch::Http::UploadedFile` to `User`:

# TODO image
And then, `ActiveStorage` will use `switch case` to choose right action to perform.

https://github.com/rails/rails/blob/master/activestorage/lib/active_storage/attached.rb#L18

## How `ActiveStorage` save file to store service and save blob record?

```ruby
def create_after_upload!(io:, filename:, content_type: nil, metadata: nil, identify: true)
  build_after_upload(io: io, filename: filename, content_type: content_type, metadata: metadata, identify: identify).tap(&:save!)
end
```

### Why `ActiveStorage` `build` before `save`?
Avoid uploading (take a long time) while having an open database transaction

### How `ActiveStorage` specify where to store file?
Inside `build_after_upload`, `ActiveStorage` will trigger upload to store service with a `key`
https://github.com/rails/rails/blob/master/activestorage/app/models/active_storage/blob.rb#L154

`key` just a token key which generated by:

```ruby
class ActiveStorage::Blob < ActiveRecord::Base
  has_secure_token :key
end
```

#### Disk
`ActiveStorage` use IO.copy_stream to upload file
- Where?

  `ActiveStorage` uses `key[0..1]/key[2..3]` folder to store file.

#### S3
`ActiveStorage` just use `aws-sdk-s3` to put file with `key`:

```ruby
def upload(key, io, checksum: nil)
  instrument :upload, key: key, checksum: checksum do
    begin
      object_for(key).put(upload_options.merge(body: io, content_md5: checksum))
    rescue Aws::S3::Errors::BadDigest
      raise ActiveStorage::IntegrityError
    end
  end
end
```

## How `ActiveStorage` generate url for a file when upload done?
`ActiveStorage` use store service to generate the `URL` for file

#### S3
```ruby
def url(key, expires_in:, filename:, disposition:, content_type:)
  ...
    generated_url = object_for(key)...

    generated_url
  ...
end
```
`S3` will handle expiration for us.

#### Disk is more complex
- How to implements `expires_in`?
When you request the `image`, it will send to `ActiveStorage::DiskController` with `params[:encoded_key]`, and `ActiveStorage` use these key to find out the key of image.
This key contains `expires_in` value, so only image which is **fresh** will be return or `ActiveStorage::DiskController` will return `:not_found`.

But what happen when show image `http://localhost:3000/users/3`, `ActiveStorage` will generate a new URL with new `expires_in`

https://github.com/rails/rails/blob/master/activestorage/app/controllers/active_storage/blobs_controller.rb#L12


For more information about how `Rails` encrypt the message with `expires_in`: https://medium.com/@michaeljcoyne/rails-5-2-metadata-options-for-messageverifier-and-messageencryptor-79540de86f9b


# Variants

`ActiveStorage` supports image transform with `ImageMagick`, just add `gem mini_magick` to application's `Gemfile`.


Processed image will be saved into `storage/va/ri/variants/:original_key/:Digest::SHA256.hexdigest(variation.key)`
https://github.com/rails/rails/blob/master/activestorage/app/models/active_storage/variant.rb#L71

`upload` and `download` are just like `Blob`

# Preview
