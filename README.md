// Import the necessary libraries
import androidx.appcompat.app.AppCompatActivity;
import androidx.camera.core.Camera;
import androidx.camera.core.CameraSelector;
import androidx.camera.core.ImageCapture;
import androidx.camera.core.ImageCaptureException;
import androidx.camera.core.Preview;
import androidx.camera.lifecycle.ProcessCameraProvider;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import android.Manifest;
import android.content.pm.PackageManager;
import android.net.Uri;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.Toast;

import com.google.common.util.concurrent.ListenableFuture;

import java.io.File;
import java.text.SimpleDateFormat;
import java.util.Locale;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

public class MainActivity extends AppCompatActivity {

    // Define the constants
    private static final String TAG = "MainActivity";
    private static final int REQUEST_CODE_PERMISSIONS = 10;
    private static final String[] REQUIRED_PERMISSIONS = new String[]{Manifest.permission.CAMERA, Manifest.permission.WRITE_EXTERNAL_STORAGE};

    // Define the views
    private PreviewView previewView;
    private Button captureButton;

    // Define the camera variables
    private ListenableFuture<ProcessCameraProvider> cameraProviderFuture;
    private ImageCapture imageCapture;
    private File outputDirectory;
    private Executor executor = Executors.newSingleThreadExecutor();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Initialize the views
        previewView = findViewById(R.id.previewView);
        captureButton = findViewById(R.id.captureButton);

        // Check if the app has the required permissions
        if (allPermissionsGranted()) {
            // Start the camera
            startCamera();
        } else {
            // Request the permissions
            ActivityCompat.requestPermissions(this, REQUIRED_PERMISSIONS, REQUEST_CODE_PERMISSIONS);
        }

        // Set the output directory
        outputDirectory = getOutputDirectory();

        // Set the click listener for the capture button
        captureButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                // Take a photo
                takePhoto();
            }
        });
    }

    // Start the camera
    private void startCamera() {
        // Get the camera provider
        cameraProviderFuture = ProcessCameraProvider.getInstance(this);

        // Add a listener to the camera provider
        cameraProviderFuture.addListener(new Runnable() {
            @Override
            public void run() {
                // Try to get the camera provider
                try {
                    ProcessCameraProvider cameraProvider = cameraProviderFuture.get();
                    // Bind the preview
                    bindPreview(cameraProvider);
                } catch (ExecutionException | InterruptedException e) {
                    // Log the error
                    Log.e(TAG, "Unable to get camera provider", e);
                }
            }
        }, ContextCompat.getMainExecutor(this));
    }

    // Bind the preview
    private void bindPreview(ProcessCameraProvider cameraProvider) {
        // Create a preview
        Preview preview = new Preview.Builder().build();

        // Create an image capture
        imageCapture = new ImageCapture.Builder().build();

        // Select the front camera
        CameraSelector cameraSelector = new CameraSelector.Builder().requireLensFacing(CameraSelector.LENS_FACING_FRONT).build();

        // Unbind any previous use cases
        cameraProvider.unbindAll();

        // Bind the preview and the image capture to the camera
        Camera camera = cameraProvider.bindToLifecycle(this, cameraSelector, preview, imageCapture);

        // Set the preview surface provider
        preview.setSurfaceProvider(previewView.getSurfaceProvider());
    }

    // Take a photo
    private void takePhoto() {
        // Check if the image capture is null
        if (imageCapture == null) {
            // Return
            return;
        }

        // Create an output file
        File photoFile = new File(outputDirectory, new SimpleDateFormat("yyyy-MM-dd-HH-mm-ss-SSS", Locale.US).format(System.currentTimeMillis()) + ".jpg");

        // Create an output options
        ImageCapture.OutputFileOptions outputFileOptions = new ImageCapture.OutputFileOptions.Builder(photoFile).build();

        // Take a picture and save it to the file
        imageCapture.takePicture(outputFileOptions, executor, new ImageCapture.OnImageSavedCallback() {
            @Override
            public void onImageSaved(ImageCapture.OutputFileResults outputFileResults) {
                // Get the saved uri
                Uri savedUri = Uri.fromFile(photoFile);

                // Display a message
                String message = "Selfie captured and saved to " + savedUri;
                Toast.makeText(MainActivity.this, message, Toast.LENGTH_SHORT).show();
                Log.d(TAG, message);
            }

            @Override
            public void onError(ImageCaptureException exception) {
                // Log the error
                Log.e(TAG, "Image capture failed: " + exception.getMessage(), exception);
            }
        });
    }

    // Get the output directory
    private File getOutputDirectory() {
        // Get the app's external files directory
        File[] mediaDirs = getExternalMediaDirs();

        // Check if the media dirs are not null and not empty
        if (mediaDirs != null && mediaDirs.length > 0) {
            // Return the first media dir
            return mediaDirs[0];
        } else {
            // Return the app's files directory
            return getFilesDir();
        }
    }

    // Check if all the permissions are granted
    private boolean allPermissionsGranted() {
        // Loop through the required permissions
        for (String permission : REQUIRED_PERMISSIONS) {
            // Check if the permission is not granted
            if (ContextCompat.checkSelfPermission(this, permission) != PackageManager.PERMISSION_GRANTED) {
                // Return false
                return false;
            }
        }
        // Return true
        return true;
    }

    // Handle the permission request result
    @Override
    public void onRequestPermissionsResult(int requestCode, String[] permissions, int[] grantResults) {
        // Check if the request code matches
        if (requestCode == REQUEST_CODE_PERMISSIONS) {
            // Check if all the permissions are granted
            if (allPermissionsGranted()) {
                // Start the camera
                startCamera();
            } else {
                // Display a message
                Toast.makeText(this, "Permissions not granted by the user.", Toast.LENGTH_SHORT).show();
                // Finish the activity
                finish();
            }
        }
    }
}
