# Integrating Cloudinary with NestJS for Image Management

- Canonical URL: https://imzihad21.github.io/articles/a/integrating-cloudinary-with-nestjs-for-image-management-25n5/
- Source URL: https://dev.to/imzihad21/integrating-cloudinary-with-nestjs-for-image-management-25n5
- Web View: https://imzihad21.github.io/articles/a/integrating-cloudinary-with-nestjs-for-image-management-25n5/
- Published: 2024-11-03T18:13:21.000Z
- Modified: 2024-11-03T18:13:21.000Z
- Reading time: 3 minutes
- Tags: nestjs, cloudinary, imageupload, backend

Image handling in backend systems is easy to start and easy to break under real traffic. A clean integration pattern with Cloudinary and NestJS helps you keep uploads stable, metadata consistent, and deletion logic predictable.

This guide shows a modular setup with provider-based configuration, upload workflows, and safe resource deletion.

### Why It Matters

- Keeps media operations isolated from business controllers.
- Stores image metadata for audit, ownership, and retrieval.
- Uses stream-based upload for memory-friendly processing.
- Makes cleanup deterministic across cloud storage and database.

### Core Concepts

#### 1. Cloudinary Provider Configuration

Create a dedicated provider that loads credentials from environment variables.

```typescript
import { Provider } from "@nestjs/common";
import { ConfigService } from "@nestjs/config";
import { v2 as CloudinaryAPI } from "cloudinary";

export const CLOUDINARY = "CLOUDINARY";

export const CloudinaryProvider: Provider = {
  provide: CLOUDINARY,
  useFactory: (configService: ConfigService) =>
    CloudinaryAPI.config({
      cloud_name: configService.get<string>("CLD_CLOUD_NAME"),
      api_key: configService.get<string>("CLD_API_KEY"),
      api_secret: configService.get<string>("CLD_API_SECRET"),
    }),
  inject: [ConfigService],
};
```

#### 2. Service Skeleton

Use a service layer to orchestrate upload and metadata persistence.

```typescript
import {
  BadRequestException,
  HttpException,
  Injectable,
  InternalServerErrorException,
  Logger,
} from "@nestjs/common";
import { ImageMetaRepository } from "./image-meta.repository";

@Injectable()
export class ImageMetaService {
  private readonly logger = new Logger(ImageMetaService.name);

  constructor(private readonly imageMetaRepository: ImageMetaRepository) {}
}
```

#### 3. Single Image Upload

Validate file input, upload to Cloudinary, then persist metadata.

```typescript
async createSingleImage(
  singleImageFile: Express.Multer.File,
  ownerId: string
): Promise<ImageMetaDocument> {
  try {
    if (!singleImageFile) {
      throw new BadRequestException("No image file provided");
    }

    const extension = this.getFileExtension(singleImageFile.originalname);
    const uploadResult = await this.uploadImageToCloudinary(singleImageFile);

    const createdImage = await this.imageMetaRepository.create({
      url: uploadResult.secure_url,
      name: uploadResult.public_id,
      extension,
      size: singleImageFile.size,
      mimeType: singleImageFile.mimetype,
      ownerId,
    });

    return createdImage;
  } catch (error) {
    this.logger.error("Error creating single image", error as Error);

    if (error instanceof HttpException) {
      throw error;
    }

    throw new InternalServerErrorException("Failed to create single image");
  }
}
```

#### 4. Multiple Image Upload

Process many files concurrently using `Promise.all`.

```typescript
async createMultipleImages(
  multipleImageFiles: Express.Multer.File[],
  ownerId: string
): Promise<ImageMetaDocument[]> {
  try {
    if (!multipleImageFiles || multipleImageFiles.length === 0) {
      throw new BadRequestException("No image files provided");
    }

    return await Promise.all(
      multipleImageFiles.map((imageFile) => this.createSingleImage(imageFile, ownerId))
    );
  } catch (error) {
    this.logger.error("Error creating multiple images", error as Error);

    if (error instanceof HttpException) {
      throw error;
    }

    throw new InternalServerErrorException("Failed to create multiple images");
  }
}
```

#### 5. Image Deletion Workflow

Delete from cloud storage and metadata store in sequence.

```typescript
async removeImage(imageId: string, ownerId: string): Promise<ImageMetaDocument> {
  try {
    const image = await this.imageMetaRepository.getOneWhere({
      _id: imageId,
      ownerId,
    });

    if (!image) {
      throw new BadRequestException(`Image not found: ${imageId}`);
    }

    await this.deleteImageFromCloudinary(image.name);
    await this.imageMetaRepository.removeOneById(imageId);

    return image;
  } catch (error) {
    if (error instanceof HttpException) {
      throw error;
    }

    this.logger.error("Error deleting image", error as Error);
    throw new InternalServerErrorException("Could not delete image");
  }
}
```

#### 6. Stream-Based Cloudinary Upload

Upload image buffers using `upload_stream` for efficient memory usage.

```typescript
import { v2 as CloudinaryAPI, UploadApiErrorResponse, UploadApiResponse } from "cloudinary";
import toStream from "buffer-to-stream";

async uploadImageToCloudinary(file: Express.Multer.File): Promise<UploadApiResponse> {
  return await new Promise<UploadApiResponse>((resolve, reject) => {
    const uploadStream = CloudinaryAPI.uploader.upload_stream(
      (error: UploadApiErrorResponse | undefined, result: UploadApiResponse | undefined) => {
        if (error) {
          reject(error);
          return;
        }

        if (!result) {
          reject(new Error("Upload result is undefined"));
          return;
        }

        resolve(result);
      }
    );

    toStream(file.buffer).pipe(uploadStream);
  });
}
```

### Practical Example

Module registration for reusable image service:

```typescript
import { Global, Module } from "@nestjs/common";
import { CloudinaryProvider } from "../../utility/provider/cloudinary.provider";
import { ImageMetaService } from "./image-meta.service";

@Global()
@Module({
  providers: [ImageMetaService, CloudinaryProvider],
  exports: [ImageMetaService],
})
export class ImageMetaModule {}
```

This gives you one place to manage uploads and deletions, so controllers do not become part-time file managers.

### Common Mistakes

- Uploading files without validating input presence and owner context.
- Saving metadata before confirming cloud upload success.
- Deleting DB records first and leaving orphaned cloud assets.
- Hardcoding Cloudinary credentials instead of environment variables.
- Ignoring concurrency and timeout behavior in bulk uploads.

### Quick Recap

- Provider pattern keeps Cloudinary configuration centralized.
- Service layer handles upload, metadata persistence, and deletion.
- `Promise.all` supports efficient multi-image workflows.
- Stream uploads improve memory behavior for file buffers.
- Modular composition keeps media logic reusable across the app.

### Next Steps

1. Add file type and size validation with `ParseFilePipe` and validators.
2. Add transformation presets (thumbnail, optimized formats) in upload options.
3. Add soft-delete metadata strategy for recovery workflows.
4. Add background cleanup jobs for failed partial operations.