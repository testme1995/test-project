RIVAL Integration in iFAST

Validation matrix

|  |  |
| --- | --- |
| Criteria | Decision |
| Service language | C++ |
| Communication protocol | gRPC |
| Callback mechanism | No |
| Cabllback endpoint | No |
| Service library | Official gRPC library |
| Input / Output | Data are on NAS |
| Security | No |

Coordinates and references

![image](https://github.com/user-attachments/assets/d0e69673-9008-4263-bede-49addc03b874)

To correctly achieve reconstruction and extraction, we will use the previous scheme as the reference. Indeed, the first slice will contain the origin (0, 0, 0) and will be the front top left slice.

General overview

For 3d reconstruction and virtual slices extraction, Dedale team will integrate the image processing recipe in a gRPC server and iFAST will create a gRPC client that requests this server by providing processing inputs and parameters.

![image](https://github.com/user-attachments/assets/253f8a72-00a8-4a75-a5b7-2eb769f8e934)

Daedalus team delivery

Daedalus team will deliver the gRPC server and a simplistic gRPC client that illustrates the processing capabilities of the server. The server will be self-contained in a docker image. For both client and server, C++ will be used.

![image](https://github.com/user-attachments/assets/958a1207-65db-4333-acce-dff9ec3e7684)

On client/server communication

Instead of waiting for the server to complete the entire task, we can configure the gRPC connection between iFast and RIVAL for streaming. Streaming enables a continuous flow of data between the client and server.

In this scenario, iFast, as the client, initiates the gRPC connection and will send continuously requests containing URI to inputs/outputs images and parameters to Rival. Moreover, the client will send at each request the temporary image path that is the location where computations are done after reconstruction and extraction.

Once the server has processed one slice, the client could send another slice and continue while the end is not reached. To finish the client should send the last slice with the explicit request to reconstruct and/or extract.

![image](https://github.com/user-attachments/assets/96da49b2-e868-407d-a0c5-06af02dcaa27)

Notice that the client **is not allowed** to perform a second request before getting a computation completion message from the server.

On server API

3d reconstruction and virtual slices extraction

For the 3d reconstruction and virtual slices extraction, we expect client and server to implement the following proto file:

syntax = "proto3";

package Rival;

service RivalService {

    // actions that can be performed by Rival Service [only sequential processing is allowed]

    rpc performAction (stream RivalRequest) returns (stream RivalStatus);

    // actions to delete directory

    rpc deleteDir (DeletionRequest) returns (DeletionResponse);

    // actions to create directory

    rpc createDir (CreationRequest) returns (CreationResponse);

    // routes used to check if the service is alive

    rpc health (HealthCheckRequest) returns (HealthCheckResponse);

    rpc watch (HealthCheckRequest) returns (stream HealthCheckResponse);

}

message CreationRequest {

    string directoryPath = 1;

}

message CreationResponse {

    enum Status {

        Failed = 0;

        Success = 1;

    }

    Status status = 1;

}

message DeletionRequest {

    string directoryPath = 1;

}

For virtual slices extraction and reconstruction, we will use only one route with different enum according to the iFast team request.

message DeletionResponse {

    enum Status {

        Failed = 0;

        Success = 1;

    }

    Status status = 1;

}

message RivalStatus {

    enum Status {

        Error = 0;

        Warning = 1;

        CommandReceived = 2;

        CommandCompleted = 3;

    }

    enum Action {

        ImageFeedForReconstruction3d = 0; // Feed images as they are acquired and do the registration for every image

        CompleteReconstruction3dAndVirtualSliceExtract = 1; // Complete reconstruction after the last image

        Reconstruction3dAndVirtualSliceExtract = 2; // Support for reconstruction with all the images acquired

        VirtualSliceExtract = 3; // Athena Use case

    }

    Status status = 1;

    Action action = 2;

    optional string errorMessage = 3;

    optional int32 UEC = 4;

    optional int32 progressPercentage = 5;

    string metric = 6;

}

message RivalRequest {

    enum Action {

        ImageFeedForReconstruction3d = 0; // Feed images as they are acquired and do the registration for every image

        CompleteReconstruction3dAndVirtualSliceExtract = 1; // Complete reconstruction after the last image

        Reconstruction3dAndVirtualSliceExtract = 2; // Support for reconstruction with all the images acquired

        VirtualSliceExtract = 3; // Athena Use case

        Abort = 4; // Is not possible now but may be later

    }

    Action action = 1;

    string outputDirectory = 2;

    // Images will be sent continuously. This will be 1 image for Action 0 and multiple images for Action 2

    optional string previousImagePath = 3;

    optional string inputImagePath = 4;

    optional StackAlignment stackAlignment = 5;

    optional Reconstruction3dParameters reconstructionParameters = 6;

    optional ExtractionParameters extractionParameters = 7;

    optional string rivalRecipe = 8;

}

message Reconstruction3dParameters {

    // shear angle in degrees

    double shearAngleInDegree = 1;

    // pixel size in nanometers

    Size pixelSize = 2;

    // milling depth in nanometers

    double millingDepth = 3;

}

message Size {

    double width = 1;

    double height = 2;

}

message BoundingBox3D {

    double MinX = 1;

    double MinY = 2;

    double MinZ = 3;

    double MaxX = 4;

    double MaxY = 5;

    double MaxZ = 6;

}

message Point2D {

    double x = 1;

    double y = 2;

}

message BoundingBox2D {

    Point2D topLeft = 1;

    Point2D bottomRight = 2;

}

message StackAlignment {

    BoundingBox2D previousFiducialBoundingBox = 1;

    BoundingBox2D currentFiducialBoundingBox = 2;

}

message ExtractionParameters {

    message SliceParameters {

        // distance from the origin

        optional double distance = 1;

        // normal of the extracted plane in pixels

        Vector3D normal = 2;

        // one arbitrary point on the slice to be extracted

        // in pixels

        Point3D origin = 3;

        // Upward direction

        Vector3D upwardDirection = 4;

        // Resolutions for width and height

        int32 resolutionU = 5;

        int32 resolutionV = 6;

        // output image path of the extracted image

        // only ASCII strings are supported

        string outputImagePath = 7;

    }

    // list of slice parameters to be extracted

    repeated SliceParameters slices = 1;

}

message Vector3D {

    double x = 1;

    double y = 2;

    double z = 3;

}

message Point3D {

    double x = 1;

    double y = 2;

    double z = 3;

}

message HealthCheckRequest {

    string service = 1;

}

message HealthCheckResponse {

    enum ServingStatus {

        UNKNOWN = 0;

        SERVING = 1;

        NOT\_SERVING = 2;

        SERVICE\_UNKNOWN = 3; // Used only by the Watch method.

    }

    ServingStatus status = 1;

}

message BoundingBox3D {

    double MinX = 1;

    double MinY = 2;

    double MinZ = 3;

    double MaxX = 4;

    double MaxY = 5;

    double MaxZ = 6;

}

message Point2D {

    double x = 1;

    double y = 2;

}

message BoundingBox2D {

    Point2D topLeft = 1;

    Point2D bottomRight = 2;

}

message StackAlignment {

    BoundingBox2D previousFiducialBoundingBox = 1;

    BoundingBox2D currentFiducialBoundingBox = 2;

}

message ExtractionParameters {

    message SliceParameters {

        // distance from the origin

        optional double distance = 1;

        // normal of the extracted plane in pixels

        Vector3D normal = 2;

        // one arbitrary point on the slice to be extracted

        // in pixels

        Point3D origin = 3;

        // Upward direction

        Vector3D upwardDirection = 4;

        // Resolutions for width and height

        int32 resolutionU = 5;

        int32 resolutionV = 6;

        // output image path of the extracted image

        // only ASCII strings are supported

        string outputImagePath = 7;

    }

    // list of slice parameters to be extracted

    repeated SliceParameters slices = 1;

}

message Vector3D {

    double x = 1;

    double y = 2;

    double z = 3;

}

message Point3D {

    double x = 1;

    double y = 2;

    double z = 3;

}

message HealthCheckRequest {

    string service = 1;

}

message HealthCheckResponse {

    enum ServingStatus {

        UNKNOWN = 0;

        SERVING = 1;

        NOT\_SERVING = 2;

        SERVICE\_UNKNOWN = 3; // Used only by the Watch method.

    }

    ServingStatus status = 1;

}

message Vector3D {

    double x = 1;

    double y = 2;

    double z = 3;

}

message Point3D {

    double x = 1;

    double y = 2;

    double z = 3;

}

message HealthCheckRequest {

    string service = 1;

}

message HealthCheckResponse {

    enum ServingStatus {

        UNKNOWN = 0;

        SERVING = 1;

        NOT\_SERVING = 2;

        SERVICE\_UNKNOWN = 3; // Used only by the Watch method.

    }

    ServingStatus status = 1;

}

On health endpoint

message HealthCheckRequest {

    string service = 1;

}

message HealthCheckResponse {

    enum ServingStatus {

        UNKNOWN = 0;

        SERVING = 1;

        NOT\_SERVING = 2;

        SERVICE\_UNKNOWN = 3; // Used only by the Watch method.

    }

    ServingStatus status = 1;

}

To enable iFAST and the orchestrator querying the health status of the service the following health route will be implemented server-side:

How to write a request in C++?

RivalClient rivalClient(grpc::CreateChannel(server\_address, grpc::InsecureChannelCredentials()));

grpc::ClientContext context;

// Create a new Reconstruction3dAndSliceExtractionRequest message

Rival::RivalRequest request;

request.set\_action(Rival::RivalRequest::Action::RivalRequest\_Action\_ImageFeedForReconstruction3d);

request.set\_temporaryimagepath("path/to/temporary.file");

// Set the values for the fields in the message

request.set\_inputimagepath("path/to/input\_image1.png");

Rival::Reconstruction3dParameters\* reconstructionParameters = request.mutable\_reconstructionparameters();

reconstructionParameters->set\_shearangleindegree(45.0);

Rival::Size\* voxelSize = reconstructionParameters->mutable\_voxelsize();

voxelSize->set\_width(1.0);

voxelSize->set\_height(1.0);

reconstructionParameters->set\_millingdepth(1000.0);

Rival::BoundingBox\* fiducialBoundingBox = reconstructionParameters->mutable\_fiducialboundingbox();

Rival::Point2D\* topLeft = fiducialBoundingBox->mutable\_topleft();

topLeft->set\_x(0.0);

topLeft->set\_y(0.0);

Rival::Point2D\* rightBottom = fiducialBoundingBox->mutable\_rightbottom();

rightBottom->set\_x(100.0);

rightBottom->set\_y(100.0);

reconstructionParameters->set\_outputimagepath("path/to/output\_image.png");

Rival::ExtractionParameters\* extractionParameters = request.mutable\_extractionparameters();

Rival::ExtractionParameters::SliceParameters\* sliceParameters = extractionParameters->add\_slices();

Rival::Vector3D\* normal = sliceParameters->mutable\_normal();

normal->set\_x(0.0);

normal->set\_y(0.0);

normal->set\_z(1.0);

Rival::Point3D\* origin = sliceParameters->mutable\_origin();

origin->set\_x(10.0);

origin->set\_y(10.0);

origin->set\_z(0.0);

sliceParameters->set\_outputimagepath("path/to/slice\_image.png");

auto reply = rivalClient.performAction(&context, &request);
