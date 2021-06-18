package com.example.dvuber;


import android.content.Intent;
import android.os.Bundle;
import android.text.TextUtils;
import android.view.LayoutInflater;
import android.view.View;
import android.widget.Button;
import android.widget.ProgressBar;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;

import com.example.dvuber.model.DriverInfoModel;
import com.firebase.ui.auth.AuthMethodPickerLayout;
import com.firebase.ui.auth.AuthUI;
import com.firebase.ui.auth.IdpResponse;

import com.google.android.material.textfield.TextInputEditText;
import com.google.firebase.auth.FirebaseAuth;
import com.google.firebase.auth.FirebaseUser;
import com.google.firebase.database.DataSnapshot;
import com.google.firebase.database.DatabaseError;
import com.google.firebase.database.DatabaseReference;
import com.google.firebase.database.FirebaseDatabase;
import com.google.firebase.database.ValueEventListener;

import java.util.Arrays;
import java.util.List;
import java.util.concurrent.TimeUnit;

import butterknife.BindViews;
import butterknife.ButterKnife;
import io.reactivex.Completable;
import io.reactivex.android.schedulers.AndroidSchedulers;


public class SplashScreenActivity extends AppCompatActivity {

    private final static int LOGING_REQUEST_CODE = 1234;
    private List<AuthUI.IdpConfig> providers;
    private FirebaseAuth firebaseAuth;
    private FirebaseAuth.AuthStateListener listener;




    @BindViews(R.id.progress_bar)
    ProgressBar progress_Bar;




    FirebaseDatabase database;
    DatabaseReference driverInfoRef;




    @Override
    protected void onStart(){
        super.onStart();
        delaySplashScreen();
    }




    @Override
    protected void onStop() {
        if (firebaseAuth != null && listener != null)
            firebaseAuth.removeAuthStateListener( listener );
        super.onStop();
    }




    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate( savedInstanceState );

        init();

    }




    protected void init() {
        ButterKnife.bind( this );


        database = FirebaseDatabase.getInstance();
                driverInfoRef = database.getReference( Common.Driver_INFO_REFERENCE);

        providers = Arrays.asList(
                new AuthUI.IdpConfig.PhoneBuilder().build(),
               new AuthUI.IdpConfig.GoogleBuilder().build());

        firebaseAuth = FirebaseAuth.getInstance();
        listener = myFirebaseAuth ->{
            FirebaseUser user = myFirebaseAuth.getCurrentUser();
            if (user != null)
            {
                checkUserFromFirebase();
            }
            else
                showLoginLayout();

        };
    }





    private void  checkUserFromFirebase(){
     driverInfoRef.child( FirebaseAuth.getInstance().getCurrentUser().getUid() )

             .addListenerForSingleValueEvent( new ValueEventListener() {
                 @Override
                 public void onDataChange(@NonNull  DataSnapshot snapshot) {
                     DataSnapshot datasnapshot = null;
                     if (datasnapshot.exists()) {
                         Toast.makeText( SplashScreenActivity.this, "user already register", Toast.LENGTH_SHORT ).show();
                     }
                     else
                     {
                       showRegisterLayout();
                     }





                 }
                 @Override
                 public void onCancelled(@NonNull  DatabaseError error) {
                     Toast.makeText( SplashScreenActivity.this, "Error", Toast.LENGTH_SHORT ).show();
                 }
             } );

}




    private void showRegisterLayout() {
            AlertDialog.Builder builder = new AlertDialog.Builder( this,R.style.DialogTheme );
                View itimView = LayoutInflater.from(this).inflate( R.layout.layout_register,null );

                TextInputEditText edt_first_name = (TextInputEditText)itimView.findViewById( R.id.edt_first_name );
                TextInputEditText edt_laste_name = (TextInputEditText)itimView.findViewById( R.id.edt_last_name );
                TextInputEditText edt_phone = (TextInputEditText)itimView.findViewById( R.id.edit_phone_number );

                Button btn_continue = (Button)itimView.findViewById( R.id.btn_register );


                //set data
                if (FirebaseAuth.getInstance().getCurrentUser().getPhoneNumber() !=null&&
                        !TextUtils.isEmpty( FirebaseAuth.getInstance().getCurrentUser().getPhoneNumber()))
                    edt_phone.setText( FirebaseAuth.getInstance().getCurrentUser().getPhoneNumber());

                builder.setView( itimView );
                        AlertDialog dialog = builder.create();
                dialog.show();

                btn_continue.setOnClickListener( v -> {
                   if (TextUtils.isEmpty( edt_first_name.getText().toString() ))
                    {
                        Toast.makeText( this, "Please enter first name ", Toast.LENGTH_SHORT ).show();
                        return;
                }
                    if (TextUtils.isEmpty( edt_laste_name.getText().toString() ))
                    {
                        Toast.makeText( this, "Please enter laste name", Toast.LENGTH_SHORT ).show();
                        return;
                    }
                    if (TextUtils.isEmpty( edt_phone.getText().toString() ))
                    {
                        Toast.makeText( this, "Please enter Phone Number", Toast.LENGTH_SHORT ).show();
                        return;
                    }
             else
                    {
                        DriverInfoModel model = new DriverInfoModel();
                                model.setFirstName( edt_first_name.getText().toString() );
                        model.setLastName( edt_laste_name.getText().toString() );
                        model.setPhoneNumber( edt_phone.getText().toString() );
                        model.setRating( 0.0 );

                        driverInfoRef.child( FirebaseAuth.getInstance().getCurrentUser().getUid() )
                                .setValue( model )
                                .addOnFailureListener( e -> Toast.makeText( SplashScreenActivity.this, e.getMessage(), Toast.LENGTH_SHORT ).show() )
                                .addOnSuccessListener( unused -> {
                                    Toast.makeText( this, "Register Succesful!", Toast.LENGTH_SHORT ).show();
                                } );
                    }

                ;
            });
    }





    protected void showLoginLayout() {
        AuthMethodPickerLayout authMethodPickerLayout = new AuthMethodPickerLayout
                .Builder( R.layout.layout_sign_in )
                .setPhoneButtonId( R.id.btn_phone_sign_in )
                .setGoogleButtonId( R.id.btn_google_sign_in )
                .build();

        startActivityForResult( AuthUI.getInstance()
                .createSignInIntentBuilder()
                .setAuthMethodPickerLayout( authMethodPickerLayout )
                .setIsSmartLockEnabled( false )
                .setTheme(R.style.LoginTheme  )
                .setAvailableProviders( providers )
                .build(), LOGING_REQUEST_CODE );
    }

    private void delaySplashScreen() {

        progress_Bar.setVisibility( View.VISIBLE );

        Completable.timer(2, TimeUnit.SECONDS,
                AndroidSchedulers.mainThread())
                .subscribe(()  ->
                        firebaseAuth.addAuthStateListener( listener )

                 );

    }





    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult( requestCode, resultCode, data );
        if(requestCode == LOGING_REQUEST_CODE)
        {
            IdpResponse response = IdpResponse.fromResultIntent( data );
            if (resultCode == RESULT_OK)
            {
                FirebaseUser user = FirebaseAuth.getInstance().getCurrentUser();
            }
            else
            {
                Toast.makeText( this, "{Error}"+response.getError().getMessage(), Toast.LENGTH_SHORT ).show();
            }
        }
    }


        }







