# Active Storage Patterns

> Lessons from 37signals' Fizzy codebase.

---

## Variant Preprocessing (#767)

Use `preprocessed: true` to prevent on-the-fly transformations failing on read replicas:

```ruby
has_many_attached :embeds do |attachable|
  attachable.variant :small,
    resize_to_limit: [800, 600],
    preprocessed: true
end
```

Centralize variant definitions in a module.

## Direct Upload Expiry (#773)

Extend expiry for slow uploads (Cloudflare buffering):

```ruby
module ActiveStorage
  mattr_accessor :service_urls_for_direct_uploads_expire_in,
    default: 48.hours
end

# Prepend to ActiveStorage::Blob
def service_url_for_direct_upload(expires_in: ActiveStorage.service_urls_for_direct_uploads_expire_in)
  super
end
```

## Large File Preview Limits (#941)

Skip previews above size threshold:

```ruby
module ActiveStorageBlobPreviewable
  MAX_PREVIEWABLE_SIZE = 16.megabytes

  def previewable?
    super && byte_size <= MAX_PREVIEWABLE_SIZE
  end
end
```

## Preview vs Variant (#770)

- **Variable** (images): `blob.variant(options)`
- **Previewable** (PDFs, videos): `blob.preview(options)`

Don't conflate them - different operations.

## Avatar Optimization (#1689)

Redirect to blob URL instead of streaming:

```ruby
def show
  if @user.avatar.attached?
    redirect_to rails_blob_url(@user.avatar.variant(:thumb))
  else
    render_initials if stale?(@user)
  end
end
```

## Mirror Configuration (#557)

Local primary + cloud backup:

```yaml
mirror:
  service: Mirror
  primary: local
  mirrors: [s3_backup]

s3_backup:
  service: S3
  force_path_style: true
  request_checksum_calculation: when_required  # For non-AWS S3
```
