# subscription


#create class
```bash


package com.example.math;

import static com.example.math.MainActivity.userEmail;

import android.content.Context;
import android.graphics.Color;
import android.util.Log;
import android.widget.TextView;

import com.android.volley.Request;
import com.android.volley.toolbox.JsonObjectRequest;
import com.android.volley.toolbox.StringRequest;
import com.android.volley.toolbox.Volley;

import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.HashMap;
import java.util.Locale;
import java.util.Map;
import java.util.concurrent.TimeUnit;

public class PremiumManager {



    public interface OnStatusFetchedListener {
        void onFetched(int status);
    }

    public static void showExpiryStatus(String expiryDate, TextView textView) {
        if (expiryDate == null || expiryDate.isEmpty() || expiryDate.equals("null")) {
            textView.setText("Status: Free User");
            return;
        }

        // যদি লাইফটাইম হয় (ডাটাবেসে যদি ৯৯৯৯ বা এমন বড় সাল থাকে)
        if (expiryDate.startsWith("9999")) {
            textView.setText("Status: Lifetime Premium 👑");
            return;
        }

        try {
            SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd", Locale.getDefault());
            Date expireDate = sdf.parse(expiryDate);
            Date today = new Date();

            long diffInMs = expireDate.getTime() - today.getTime();
            long daysLeft = TimeUnit.MILLISECONDS.toDays(diffInMs);

            if (daysLeft > 0) {
                textView.setText("Premium expires in: " + daysLeft + " days");
            } else {
                textView.setText("Status: Membership Expired");
            }
        } catch (Exception e) {
            textView.setText("Status: Active");
        }
    }




    // ১. মেম্বারশিপ চেক করার মেইন মেথড
    public static void checkUserMembership(Context context, OnStatusFetchedListener listener) {
        if (MainActivity.userEmail == null || MainActivity.userEmail.isEmpty()) return;

        String url = context.getString(R.string.Base_url) + "login.php?get_user_info=true&email=" + MainActivity.userEmail;

        JsonObjectRequest request = new JsonObjectRequest(Request.Method.GET, url, null,
                response -> {
                    try {
                        if (response.optString("status").equals("success")) {

                            // প্রিমিয়াম স্ট্যাটাস রিসিভ (১, ২ বা ৩)
                            int pStatus = response.optInt("premium_status", 0);

                            // MainActivity-র ভেরিয়েবল আপডেট
                            MainActivity.premiumStatus = pStatus;
                            MainActivity.isPremiumActive = (pStatus > 0);

                            // যদি লিসেনার থাকে তবে তাকে জানানো (UI আপডেট করার জন্য)
                            if (listener != null) {
                                listener.onFetched(pStatus);
                            }

                            Log.d("PremiumManager", "Status Sync Success: " + pStatus);
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }, error -> Log.e("Volley", "Error: " + error.getMessage()));

        Volley.newRequestQueue(context).add(request);
    }

    public static void updatePremiumOnServer(Context context, String email, int status, OnUpdateListener listener) {
        String url = context.getString(R.string.Base_url) + "login.php";

        StringRequest request = new StringRequest(Request.Method.POST, url,
                response -> {
                    try {
                        android.util.Log.d("PremiumManager", "Response: " + response);
                        org.json.JSONObject jsonObject = new org.json.JSONObject(response);

                        // যদি স্ট্যাটাস সাকসেস হয়, তবে শুধু লিসেনারকে কল করবে
                        if (jsonObject.getString("status").equals("success")) {
                            if (listener != null) {
                                listener.onUpdateSuccess();
                            }
                        }
                    } catch (org.json.JSONException e) {
                        e.printStackTrace();
                        android.util.Log.e("PremiumManager", "JSON Error: " + e.getMessage());
                    }
                },
                error -> android.util.Log.e("PremiumManager", "Volley Error: " + error.getMessage())) {

            @Override
            protected java.util.Map<String, String> getParams() {
                java.util.Map<String, String> params = new java.util.HashMap<>();
                params.put("action", "update_premium");
                params.put("email", email);
                params.put("package_type", String.valueOf(status));
                return params;
            }
        };
        com.android.volley.toolbox.Volley.newRequestQueue(context).add(request);
    }

    public interface OnUpdateListener {
        void onUpdateSuccess();
    }

    // ২. রিওয়ার্ড ক্যালকুলেট করা (Daily Reward দ্বিগুণ)
    public static int getRewardAmount(int baseAmount) {
        if (MainActivity.isPremiumActive) {
            return baseAmount * 2; // প্রিমিয়াম হলে ২ গুণ
        }
        return baseAmount;
    }

    // ৩. নামের পাশে ক্রাউন বা মুকুট সেট করা
    public static void applyPremiumName(TextView textView, String originalName) {
        if (MainActivity.isPremiumActive) {
            textView.setText(originalName + " 👑");
            textView.setTextColor(Color.parseColor("#FFD600")); // গোল্ডেন কালার
        } else {
            textView.setText(originalName);
        }
    }



}

```

#PREMIMUM DILOG CLASS CODE

  ```bash

    public interface OnPremiumListener {
        void onActivated(int packageType);
    }



    public void showPremiumDialog(String userEmail, OnPremiumListener listener) {
        // ১. সার্ভার থেকে লেটেস্ট স্ট্যাটাস চেক
        PremiumManager.checkUserMembership(context, currentStatus -> {

            Dialog dialog = new Dialog(context);
            dialog.setContentView(R.layout.dialog_premium);

            if (dialog.getWindow() != null) {
                dialog.getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
                dialog.getWindow().setLayout(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.WRAP_CONTENT);
            }

            // --- View Binding ---
            com.google.android.material.card.MaterialCardView planBasic = dialog.findViewById(R.id.planBasic);
            com.google.android.material.card.MaterialCardView planPro   = dialog.findViewById(R.id.planPro);
            com.google.android.material.card.MaterialCardView planLife  = dialog.findViewById(R.id.planLife);

            Button btnSubscribe = dialog.findViewById(R.id.btnSubscribe);
            TextView tvClose    = dialog.findViewById(R.id.tvClose);



            final int[] selectedPackage = {currentStatus > 0 ? currentStatus : 2};

            View.OnClickListener selectionListener = v -> {
                // কার্ড রিসেট
                planBasic.setStrokeWidth(0);
                planPro.setStrokeWidth(0);
                planLife.setStrokeWidth(0);

                int id = v.getId();
                if (id == R.id.planBasic) {
                    selectedPackage[0] = 1;
                    planBasic.setStrokeWidth(5);
                    planBasic.setStrokeColor(Color.parseColor("#FFD600"));
                } else if (id == R.id.planPro) {
                    selectedPackage[0] = 2;
                    planPro.setStrokeWidth(5);
                    planPro.setStrokeColor(Color.parseColor("#FFD600"));
                } else if (id == R.id.planLife) {
                    selectedPackage[0] = 3;
                    planLife.setStrokeWidth(5);
                    planLife.setStrokeColor(Color.parseColor("#FFD600"));
                }



                // বাটন টেক্সট ও স্টেট আপডেট
                if (selectedPackage[0] == currentStatus) {
                    btnSubscribe.setText("CURRENTLY ACTIVE");
                    btnSubscribe.setTextColor(Color.RED);
                    btnSubscribe.setEnabled(false);
                    btnSubscribe.setAlpha(0.5f);
                } else {
                    btnSubscribe.setEnabled(true);
                    btnSubscribe.setAlpha(1.0f);
                    btnSubscribe.setText("UPGRADE NOW");

                    btnSubscribe.setOnClickListener(view -> {
                        PremiumManager.updatePremiumOnServer(context, userEmail, selectedPackage[0], () -> {
                            dialog.dismiss();
                            if (listener != null) listener.onActivated(selectedPackage[0]);
                        });
                    });
                }
            };

            planBasic.setOnClickListener(selectionListener);
            planPro.setOnClickListener(selectionListener);
            planLife.setOnClickListener(selectionListener);

            // শুরুতে বর্তমান প্যাকেজটি ট্রিগার করা
            if (currentStatus == 1) planBasic.performClick();
            else if (currentStatus == 3) planLife.performClick();
            else planPro.performClick();

            tvClose.setOnClickListener(v -> dialog.dismiss());
            dialog.show();
        });
    }




```

# ads manger premimum code

```bash

package com.example.math;

import android.app.Activity;
import android.content.Context;
import android.os.Handler;
import android.util.Log;
import android.view.View;
import android.widget.LinearLayout;
import androidx.annotation.NonNull;

import com.android.volley.Request;
import com.android.volley.toolbox.StringRequest;
import com.android.volley.toolbox.Volley;
import com.google.android.gms.ads.AdRequest;
import com.google.android.gms.ads.AdSize;
import com.google.android.gms.ads.AdView;
import com.google.android.gms.ads.FullScreenContentCallback;
import com.google.android.gms.ads.LoadAdError;
import com.google.android.gms.ads.MobileAds;
import com.google.android.gms.ads.interstitial.InterstitialAd;
import com.google.android.gms.ads.interstitial.InterstitialAdLoadCallback;
import com.google.android.gms.ads.rewarded.RewardedAd;
import com.google.android.gms.ads.rewarded.RewardedAdLoadCallback;

import org.json.JSONObject;

import java.util.HashMap;
import java.util.Map;

public class AdsManager {

    private static final String TAG = "AdsManagerLog";

    // ✅ Singleton Instance
    private static AdsManager instance;

    public static String interstitialId = "";
    public static String rewardedId = "";
    public static String bannerId = "";

    public static String bannerStatus = "off";
    public static String interStatus = "off";
    public static String videoStatus = "off";

    private InterstitialAd mInterstitialAd;
    private RewardedAd mRewardedAd;
    private AdView mAdView;
    private Context context;

    private boolean shouldShowImmediately = false;

    // ✅ Private Constructor
    public AdsManager(Context context) {
        this.context = context.getApplicationContext();
        MobileAds.initialize(this.context, initializationStatus -> {
            Log.d(TAG, "AdMob Initialized");
        });
    }

    // ✅ getInstance
    public static AdsManager getInstance(Context context) {
        if (instance == null) {
            instance = new AdsManager(context.getApplicationContext());
        }
        return instance;
    }

    // ✅ init: প্রিমিয়াম চেক করে অ্যাড লোড করবে
    public void init(Activity activity) {
        fetchConfig(activity, () -> {
            // ১. সরাসরি PremiumManager থেকে স্ট্যাটাস চেক করে ডিসিশন নেওয়া হচ্ছে
            PremiumManager.checkUserMembership(activity, status -> {
                MainActivity.premiumStatus = status; // ডাটা সিঙ্ক করে রাখা হলো

                if (status <= 0) { // যদি প্রিমিয়াম না হয়
                    activity.runOnUiThread(() -> {
                        loadInterstitial(activity);
                        loadRewarded(activity);
                    });
                } else {
                    Log.d(TAG, "Premium User: Ads loading skipped.");
                }
            });
        });
    }

    // --- মেথড আপডেট: সরাসরি প্রিমিয়াম স্ট্যাটাস চেক করবে ---
    public boolean isPremiumUser() {
        return MainActivity.premiumStatus > 0;
    }

    public static void fetchConfig(Context context, final AdsFetchListener listener) {
        String url = context.getString(R.string.Base_url) + "get_ads.php";
        StringRequest stringRequest = new StringRequest(Request.Method.GET, url,
                response -> {
                    try {
                        JSONObject jsonObject = new JSONObject(response);
                        bannerId = jsonObject.getString("banner_id");
                        interstitialId = jsonObject.getString("inter_id");
                        rewardedId = jsonObject.getString("video_id");

                        bannerStatus = jsonObject.getString("banner_status");
                        interStatus = jsonObject.getString("inter_status");
                        videoStatus = jsonObject.getString("video_status");

                        listener.onComplete();
                    } catch (Exception e) {
                        listener.onComplete();
                    }
                },
                error -> listener.onComplete()
        );
        Volley.newRequestQueue(context).add(stringRequest);
    }

    public interface AdsFetchListener {
        void onComplete();
    }

    public void showBannerAd(LinearLayout adContainer) {
        // ডিসিশন: প্রিমিয়াম ইউজার হলে অ্যাড দেখাবে না
        if (isPremiumUser() || bannerStatus.equals("off") || bannerId.isEmpty()) {
            if (adContainer != null) adContainer.setVisibility(View.GONE);
            return;
        }
        mAdView = new AdView(context);
        mAdView.setAdUnitId(bannerId);
        mAdView.setAdSize(AdSize.BANNER);
        adContainer.removeAllViews();
        adContainer.addView(mAdView);
        mAdView.loadAd(new AdRequest.Builder().build());
        adContainer.setVisibility(View.VISIBLE);
    }

    public void loadInterstitial(Activity activity) {
        if (isPremiumUser() || interstitialId.isEmpty()) return;

        AdRequest adRequest = new AdRequest.Builder().build();
        InterstitialAd.load(context, interstitialId, adRequest, new InterstitialAdLoadCallback() {
            @Override
            public void onAdLoaded(@NonNull InterstitialAd interstitialAd) {
                mInterstitialAd = interstitialAd;
                if (shouldShowImmediately && activity != null) {
                    showAdAlways(activity, "inter", () -> {});
                    shouldShowImmediately = false;
                }
            }
            @Override
            public void onAdFailedToLoad(@NonNull LoadAdError loadAdError) {
                mInterstitialAd = null;
                new Handler().postDelayed(() -> loadInterstitial(null), 15000);
            }
        });
    }

    public void loadRewarded(Activity activity) {
        if (isPremiumUser() || rewardedId.isEmpty()) return;

        AdRequest adRequest = new AdRequest.Builder().build();
        RewardedAd.load(context, rewardedId, adRequest, new RewardedAdLoadCallback() {
            @Override
            public void onAdLoaded(@NonNull RewardedAd rewardedAd) {
                mRewardedAd = rewardedAd;
                if (shouldShowImmediately && activity != null) {
                    showAdAlways(activity, "video", () -> {});
                    shouldShowImmediately = false;
                }
            }
            @Override
            public void onAdFailedToLoad(@NonNull LoadAdError loadAdError) {
                mRewardedAd = null;
                new Handler().postDelayed(() -> loadRewarded(null), 15000);
            }
        });
    }

    public void loadInterstitial() { loadInterstitial(null); }
    public void loadRewarded() { loadRewarded(null); }

    public void showAd(Activity activity, String type, Runnable onAdDone) {
        if (isPremiumUser()) { if (onAdDone != null) onAdDone.run(); return; }

        if (type.equals("inter") && interStatus.equals("on")) {
            showAdAlways(activity, "inter", onAdDone);
        } else if (type.equals("video") && videoStatus.equals("on")) {
            showAdAlways(activity, "video", onAdDone);
        } else {
            if (onAdDone != null) onAdDone.run();
        }
    }

    public void showAdAlways(Activity activity, String type, Runnable onAdDone) {
        if (isPremiumUser()) { if (onAdDone != null) onAdDone.run(); return; }

        if (type.equals("inter")) {
            if (mInterstitialAd != null) {
                mInterstitialAd.setFullScreenContentCallback(new FullScreenContentCallback() {
                    @Override
                    public void onAdDismissedFullScreenContent() {
                        mInterstitialAd = null;
                        loadInterstitial(null);
                        if (onAdDone != null) onAdDone.run();
                    }
                    @Override
                    public void onAdFailedToShowFullScreenContent(@NonNull com.google.android.gms.ads.AdError adError) {
                        mInterstitialAd = null;
                        loadInterstitial(null);
                        if (onAdDone != null) onAdDone.run();
                    }
                });
                mInterstitialAd.show(activity);
            } else {
                shouldShowImmediately = true;
                loadInterstitial(activity);
                if (onAdDone != null) onAdDone.run();
            }
        } else if (type.equals("video")) {
            if (mRewardedAd != null) {
                mRewardedAd.setFullScreenContentCallback(new FullScreenContentCallback() {
                    @Override
                    public void onAdDismissedFullScreenContent() {
                        mRewardedAd = null;
                        loadRewarded(null);
                        if (onAdDone != null) onAdDone.run();
                    }
                    @Override
                    public void onAdFailedToShowFullScreenContent(@NonNull com.google.android.gms.ads.AdError adError) {
                        mRewardedAd = null;
                        loadRewarded(null);
                        if (onAdDone != null) onAdDone.run();
                    }
                });
                mRewardedAd.show(activity, rewardItem -> Log.d(TAG, "Rewarded Success"));
            } else {
                shouldShowImmediately = true;
                loadRewarded(activity);
                if (onAdDone != null) onAdDone.run();
            }
        } else {
            if (onAdDone != null) onAdDone.run();
        }
    }
}



```

# setting fregment code 

```bash


package com.example.math.Fragment;

import static android.content.Context.MODE_PRIVATE;

import android.animation.AnimatorSet;
import android.animation.ObjectAnimator;
import android.animation.ValueAnimator;
import android.content.SharedPreferences;
import android.content.res.Configuration;
import android.content.res.Resources;
import android.graphics.Color;
import android.os.Bundle;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.view.animation.AccelerateDecelerateInterpolator;
import android.view.animation.OvershootInterpolator;
import android.widget.RelativeLayout;
import android.widget.TextView;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.appcompat.app.AlertDialog;
import androidx.fragment.app.Fragment;

import com.example.math.DialogBox;
import com.example.math.MainActivity;
import com.example.math.R;
import com.google.android.material.card.MaterialCardView;

import java.util.Locale;

public class SettingFragment extends Fragment {


    private TextView tvCurrentLang,tvExpiry; // বর্তমান ভাষা দেখানোর জন্য

    // ২০টি ভাষার লিস্ট
    private final String[] langNames = {
            "🇺🇸 English", "🇧🇩 Bengali", "🇮🇳 Hindi", "🇪🇸 Spanish", "🇫🇷 French",
            "🇸🇦 Arabic", "🇵🇹 Portuguese", "🇷🇺 Russian", "🇯🇵 Japanese", "🇩🇪 German",
            "🇨🇳 Chinese", "🇰🇷 Korean", "🇮🇹 Italian", "🇹🇷 Turkish", "🇻湳 Vietnamese",
            "🇹🇭 Thai", "🇮🇩 Indonesian", "🇳🇱 Dutch", "🇵🇱 Polish", "🇵🇭 Tagalog"
    };

    private final String[] langCodes = {
            "en", "bn", "hi", "es", "fr", "ar", "pt", "ru", "ja", "de",
            "zh", "ko", "it", "tr", "vi", "th", "id", "nl", "pl", "tl"
    };
    // DialogBox এর অবজেক্ট (আপনার ক্লাসের কনটেক্সট অনুযায়ী)
    DialogBox dialogBox;

    MaterialCardView cardPremium;
    TextView btnUpgrade,crownIcon;
   RelativeLayout rowLanguage;


    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater,
                             @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {

        // অ্যাপ শুরুতেই সেভ করা ভাষা অনুযায়ী কনফিগারেশন সেট করা
        loadLocale();

        return inflater.inflate(R.layout.fragment_setting, container, false);
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        dialogBox = new DialogBox(requireActivity());
        init(view);
        // বর্তমানে কোন ভাষা সেট করা আছে তা TextView-তে দেখানো
        updateCurrentLangText();

        // ── Crown animation ──────────────────────────────────────────────────
        startCrownAnimation();

        clickListener();







    }//////***********************END ON CREAATE**********************************/////////////////////////





    public void init(View view){
        tvCurrentLang = view.findViewById(R.id.tvCurrentLang);
        crownIcon = view.findViewById(R.id.crownIcon);
        cardPremium = view.findViewById(R.id.cardPremium);
        btnUpgrade = view.findViewById(R.id.btnUpgrade);
        crownIcon = view.findViewById(R.id.crownIcon);
        rowLanguage = view.findViewById(R.id.rowLanguage);
        tvExpiry = view.findViewById(R.id.tvExpiryDate);

    }
    public void clickListener(){

        rowLanguage.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                animatePress(v);
                showLanguageDialog();
            }
        });




        cardPremium.setOnClickListener(v -> {
            // ১. MainActivity থেকে ইউজারের ইমেইল নেওয়া (নিশ্চিত করুন userEmail স্ট্যাটিক)
            String userEmail = MainActivity.userEmail;

            // ২. ইমেইল চেক (null চেকসহ)
            if (userEmail == null || userEmail.isEmpty()) {
                if (getContext() != null) {
                    android.widget.Toast.makeText(getContext(), "Please login first!", android.widget.Toast.LENGTH_SHORT).show();
                }
                return;
            }

            // ৩. ডায়ালগ বক্স ওপেন (নিশ্চিত করুন dialogBox অবজেক্টটি ইনিশিয়ালাইজ করা আছে)
            // উদাহরণ: DialogBox dialogBox = new DialogBox(getContext());
            dialogBox.showPremiumDialog(userEmail, new DialogBox.OnPremiumListener() {
                @Override
                public void onActivated(int pType) {

                    // ৪. সার্ভারে আপডেট পাঠানো
                    PremiumManager.updatePremiumOnServer(getContext(), userEmail, pType, new PremiumManager.OnUpdateListener() {
                        @Override
                        public void onUpdateSuccess() {

                            // ৫. UI রিফ্রেশ (কার্ড গোল্ডেন করা)
                            updatePremiumUI();

                            // ৬. অভিনন্দন মেসেজ দেখানো
                            if (getActivity() != null) {
                                new android.app.AlertDialog.Builder(getActivity())
                                        .setTitle("Congratulations! 👑")
                                        .setMessage("Your Premium Membership is now Active!\nRoom Cards have been added to your account.")
                                        .setCancelable(false)
                                        .setPositiveButton("Awesome!", (dialog, which) -> {
                                            dialog.dismiss();
                                            android.widget.Toast.makeText(getContext(), "Profile Updated!", android.widget.Toast.LENGTH_SHORT).show();
                                        })
                                        .show();
                            }
                        }
                    });
                }
            });
        });




    }


    private void updatePremiumUI() {
        // ইউজারের ইমেইল না থাকলে কিছুই করার দরকার নেই
        if (MainActivity.userEmail == null || MainActivity.userEmail.isEmpty()) return;

        // ১. সার্ভার থেকে ডাটা আসার আগেই লোকাল ভেরিয়েবল চেক করে ডিজাইন সেট করে দিন
        // এতে ডিজাইন হুট করে গায়েব হবে না
        if (MainActivity.premiumStatus > 0) {
            setGoldenDesign();
        } else {
            setWhiteDesign();
        }

        // ২. এবার ব্যাকগ্রাউন্ডে সার্ভার থেকে লেটেস্ট স্ট্যাটাস চেক করুন (যাতে মেয়াদ শেষ হলে আপডেট হয়)
        PremiumManager.checkUserMembership(requireContext(), (statusFromServer) -> {
            // যদি সার্ভারের স্ট্যাটাস লোকাল স্ট্যাটাসের চেয়ে আলাদা হয়, তবেই ডিজাইন চেঞ্জ হবে
            if (statusFromServer > 0) {
                setGoldenDesign();
            } else {
                setWhiteDesign();
            }
        });
    }

    // ডিজাইন বারবার না লিখে আলাদা মেথড করে নিন (কোড ক্লিন থাকবে)
    private void setGoldenDesign() {
        // ১. কার্ডের ডিজাইন সেট করা (গোল্ডেন থিম)
        cardPremium.setCardBackgroundColor(Color.parseColor("#FFD600"));
        cardPremium.setStrokeColor(Color.parseColor("#E6C200"));
        cardPremium.setStrokeWidth(4);

        // ২. বাটনের টেক্সট এবং স্টাইল আপডেট
        btnUpgrade.setText("ACTIVED 👑"); // সেমিকোলন ফিক্স করা হয়েছে
        btnUpgrade.setEnabled(false);
        btnUpgrade.setAlpha(0.9f);

        if (tvExpiry != null) {
            tvExpiry.setText("Checking status...");

            // ১. আপনার আগের মেথড কল করুন (এটি স্ট্যাটাস চেক করবে)
            PremiumManager.checkUserMembership(getActivity(), (status) -> {

                if (status > 0) {
                    // ২. যেহেতু আপনার লিসেনারে টাইম নেই, তাই আলাদাভাবে ডাটাটি টেনে আনছি
                    String url = getString(R.string.Base_url) + "login.php?get_user_info=true&email=" + MainActivity.userEmail;

                    com.android.volley.toolbox.JsonObjectRequest request = new com.android.volley.toolbox.JsonObjectRequest(
                            com.android.volley.Request.Method.GET, url, null,
                            response -> {
                                // ৩. PHP থেকে আসা 'premium_expiry' (যেটাতে আমরা সময় সাজিয়ে পাঠিয়েছি) সরাসরি সেট করুন
                                String timeLeft = response.optString("premium_expiry", "Active");
                                tvExpiry.setText("Time Left : "+timeLeft);
                            },
                            error -> tvExpiry.setText("Premium Active")
                    );
                    com.android.volley.toolbox.Volley.newRequestQueue(getActivity()).add(request);

                } else {
                    tvExpiry.setText("Status: Free User");
                }
            });
        }


    }

    private void setWhiteDesign() {
        cardPremium.setCardBackgroundColor(Color.WHITE);
        cardPremium.setStrokeWidth(0);
        btnUpgrade.setText("UPGRADE TO PREMIUM");
        btnUpgrade.setEnabled(true);
        btnUpgrade.setAlpha(1.0f);
    }

    @Override
    public void onResume() {
        super.onResume();
        updatePremiumUI();
    }

    // ── Language Methods ──────────────────────────────────────────────────

    private void showLanguageDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(requireContext());
        builder.setTitle(R.string.st_language); // strings.xml থেকে টাইটেল নেবে
        builder.setItems(langNames, (dialog, which) -> {
            setLocale(langCodes[which]);
        });
        builder.show();
    }

    private void setLocale(String langCode) {
        SharedPreferences preferences = requireActivity().getSharedPreferences("Settings", MODE_PRIVATE);
        preferences.edit().putString("My_Lang", langCode).apply();

        // অ্যাপটি পুরোপুরি রিস্টার্ট করার প্রফেশনাল উপায়
        android.content.Intent i = requireActivity().getBaseContext().getPackageManager()
                .getLaunchIntentForPackage(requireActivity().getBaseContext().getPackageName());
        i.addFlags(android.content.Intent.FLAG_ACTIVITY_CLEAR_TOP);
        i.addFlags(android.content.Intent.FLAG_ACTIVITY_NEW_TASK);
        requireActivity().finish();
        startActivity(i);
    }


    private void loadLocale() {
        SharedPreferences preferences = requireActivity().getSharedPreferences("Settings", MODE_PRIVATE);
        String language = preferences.getString("My_Lang", "en");
        updateResources(language);
    }

    private void updateResources(String langCode) {
        Locale locale = new Locale(langCode);
        Locale.setDefault(locale);

        Resources res = requireContext().getResources();
        Configuration config = new Configuration(res.getConfiguration());

        // আধুনিক পদ্ধতি
        config.setLocale(locale);
        requireContext().createConfigurationContext(config); // নতুন কনটেক্সট তৈরি
        res.updateConfiguration(config, res.getDisplayMetrics());
    }

    private void updateCurrentLangText() {
        SharedPreferences preferences = requireActivity().getSharedPreferences("Settings", MODE_PRIVATE);
        String currentCode = preferences.getString("My_Lang", "en");
        for (int i = 0; i < langCodes.length; i++) {
            if (langCodes[i].equals(currentCode)) {
                tvCurrentLang.setText(langNames[i]);
                break;
            }
        }
    }

    // ── Delete Account Dialog ─────────────────────────────────────────────


    // ── Crown Animation & animatePress (আপনার আগের কোডটিই থাকবে) ──────────────────
    private void startCrownAnimation() {
        if (crownIcon == null) return;
        ObjectAnimator spinIn = ObjectAnimator.ofFloat(crownIcon, "rotation", 0f, 360f);
        spinIn.setDuration(700);
        spinIn.setInterpolator(new OvershootInterpolator(1.5f));
        ObjectAnimator wobble = ObjectAnimator.ofFloat(crownIcon, "rotation", -12f, 12f, -8f, 8f, 0f);
        wobble.setDuration(1800);
        wobble.setInterpolator(new AccelerateDecelerateInterpolator());
        wobble.setRepeatCount(ValueAnimator.INFINITE);
        wobble.setRepeatMode(ValueAnimator.RESTART);
        wobble.setStartDelay(2000);
        AnimatorSet scaleUp = new AnimatorSet();
        scaleUp.playTogether(ObjectAnimator.ofFloat(crownIcon, "scaleX", 1f, 1.25f), ObjectAnimator.ofFloat(crownIcon, "scaleY", 1f, 1.25f));
        scaleUp.setDuration(180);
        AnimatorSet scaleDown = new AnimatorSet();
        scaleDown.playTogether(ObjectAnimator.ofFloat(crownIcon, "scaleX", 1.25f, 1f), ObjectAnimator.ofFloat(crownIcon, "scaleY", 1.25f, 1f));
        scaleDown.setDuration(180);
        spinIn.addListener(new android.animation.AnimatorListenerAdapter() {
            public void onAnimationEnd(android.animation.Animator a) {
                scaleUp.start();
                scaleUp.addListener(new android.animation.AnimatorListenerAdapter() {
                    public void onAnimationEnd(android.animation.Animator a) {
                        scaleDown.start();
                        scaleDown.addListener(new android.animation.AnimatorListenerAdapter() {
                            public void onAnimationEnd(android.animation.Animator a) {
                                wobble.start();
                            }
                        });
                    }
                });
            }
        });
        crownIcon.postDelayed(spinIn::start, 400);
    }

    private void animatePress(View v) {
        AnimatorSet press = new AnimatorSet();
        press.playTogether(ObjectAnimator.ofFloat(v, "scaleX", 1f, 0.96f), ObjectAnimator.ofFloat(v, "scaleY", 1f, 0.96f));
        press.setDuration(80);
        AnimatorSet release = new AnimatorSet();
        release.playTogether(ObjectAnimator.ofFloat(v, "scaleX", 0.96f, 1f), ObjectAnimator.ofFloat(v, "scaleY", 0.96f, 1f));
        release.setDuration(150);
        release.setInterpolator(new OvershootInterpolator());
        AnimatorSet full = new AnimatorSet();
        full.playSequentially(press, release);
        full.start();
    }
    
    
    
    
}


```


# daily claim 2x

```bash


    private void handleClaim(android.widget.Button button, String trophyAmount, String taskType) {
        if (!isAdded() || getContext() == null) return;

        // --- ১. প্রিমিয়াম লজিক প্রয়োগ: ২ গুণ রিওয়ার্ড ---
        // trophyAmount String হিসেবে আছে, তাই প্রথমে Integer এ কনভার্ট করে PremiumManager দিয়ে ডাবল করতে হবে
        int baseAmount = Integer.parseInt(trophyAmount);
        int finalAmount = PremiumManager.getRewardAmount(baseAmount); // প্রিমিয়াম হলে ২ গুণ হবে
        String finalTrophyStr = String.valueOf(finalAmount);

        // লোডিং দেখানো
        DialogBox loading = new DialogBox(requireContext());
        loading.showLoading("Claiming Reward", "Please wait...");

        String url = getString(R.string.Base_url) + "claim_reward.php";
        String email = MainActivity.userEmail;

        java.text.SimpleDateFormat sdf = new java.text.SimpleDateFormat("yyyy-MM-dd", java.util.Locale.getDefault());
        String currentDate = sdf.format(new java.util.Date());

        StringRequest request = new StringRequest(Request.Method.POST, url,
                response -> {
                    if (!isAdded()) return;
                    loading.hideLoading();

                    try {
                        String cleanResponse = response.trim();
                        org.json.JSONObject json = new org.json.JSONObject(cleanResponse);

                        if (json.optString("status").equals("success")) {
                            setButtonClaimed(button);

                            int streak = json.optInt("streak", 1);

                            // ২. সাকসেস মেসেজেও দ্বিগুণ অ্যামাউন্ট দেখানো (finalTrophyStr ব্যবহার করা হয়েছে)
                            loading.showStreakDialog(streak, "Success! You got " + finalTrophyStr + " 🏆");

                            loadDailyProgress();
                            fetchUserDataFromServer(email);

                        } else {
                            String msg = json.optString("message");
                            if (msg.contains("Already")) setButtonClaimed(button);
                            android.widget.Toast.makeText(getContext(), msg, android.widget.Toast.LENGTH_SHORT).show();
                        }
                    } catch (Exception e) {
                        android.util.Log.e("CLAIM_ERROR", "JSON Parse Error: " + response);
                        android.widget.Toast.makeText(getContext(), "Server error, try again!", android.widget.Toast.LENGTH_SHORT).show();
                    }
                },
                error -> {
                    if (isAdded()) {
                        loading.hideLoading();
                        android.widget.Toast.makeText(getContext(), "Network Error!", android.widget.Toast.LENGTH_SHORT).show();
                    }
                }) {

            @Override
            protected java.util.Map<String, String> getParams() {
                java.util.HashMap<String, String> params = new java.util.HashMap<>();
                params.put("email", email != null ? email : "");

                // ৩. সার্ভারে দ্বিগুণ অ্যামাউন্ট পাঠানো হচ্ছে
                params.put("trophy_to_add", finalTrophyStr);

                params.put("task_type", taskType);
                params.put("date", currentDate);
                return params;
            }
        };

        request.setRetryPolicy(new com.android.volley.DefaultRetryPolicy(15000, 0, 1.0f));
        request.setShouldCache(false);
        com.android.volley.toolbox.Volley.newRequestQueue(requireContext()).add(request);
    }




```


#claim_reward.php


```bash

ay php file cahnge na korle o hobe..

```



#login.php


```bash

<?php
header('Content-Type: application/json');
require 'config.php'; 

if (!isset($conn)) {
    echo json_encode(["status" => "error", "message" => "Database connection variable not found"]);
    exit();
}

// --- ১. GET Request: ডাটা রিড এবং লাইভ সময় (Min/Sec) ক্যালকুলেশন ---
if ($_SERVER['REQUEST_METHOD'] === 'GET' && isset($_GET['get_user_info']) && isset($_GET['email'])) {
    $email = mysqli_real_escape_string($conn, $_GET['email']);
    
    $result = mysqli_query($conn, "SELECT * FROM users WHERE email='$email'");
    
    if ($row = mysqli_fetch_assoc($result)) {
        
        $user_id = $row['id'];
        $p_status = (int)$row['premium_status'];
        $p_expiry = $row['premium_expiry'];
        $formatted_time = ""; // জাভা অ্যাপে দেখানোর জন্য তৈরি করা স্ট্রিং

        // --- প্রিমিয়াম মেয়াদ চেক এবং সময় গণনা লজিক ---
        if ($p_status > 0 && !empty($p_expiry)) {
            $current_time = new DateTime();
            $expiry_time = new DateTime($p_expiry);

            if ($expiry_time < $current_time) {
                // মেয়াদ শেষ হলে ডাটাবেস আপডেট
                mysqli_query($conn, "UPDATE users SET premium_status = 0 WHERE id = '$user_id'");
                $p_status = 0;
                $formatted_time = "Expired";
            } else {
                // প্যাকেজ ৩ বা সাল ৯৯৯৯ হলে লাইফটাইম
                if ($p_status == 3 || strpos($p_expiry, '9999') !== false) {
                    $formatted_time = "Lifetime Premium 👑";
                } else {
                    // দিন, ঘণ্টা, মিনিট এবং সেকেন্ডের পার্থক্য বের করা
                    $diff = $current_time->diff($expiry_time);
                    $formatted_time = $diff->format('%a Days %h Hrs %i Min %s Sec');
                }
            }
        } else {
            $formatted_time = "Free Plan";
        }

        echo json_encode([
            "status" => "success",
            "user_id" => (int)$row['id'],
            "name" => $row['name'],
            "photo" => $row['photo'],
            "points" => (int)$row['points'], 
            "tokens" => (int)$row['tokens'],
            "trophy" => (int)$row['trophy'],
            "room_cards" => (int)($row['room_cards'] ?? 0),
            "level" => (int)($row['level'] ?? 1),
            "xp" => (int)($row['xp'] ?? 0),
            "perfect_wins" => (int)($row['perfect_wins'] ?? 0),
            "games_won" => (int)($row['games_won'] ?? 0),
            "premium_status" => $p_status,
            "premium_expiry" => $formatted_time // ✅ ট্রিক: expiry-র জায়গায় আমরা সাজানো টেক্সটটি পাঠিয়ে দিচ্ছি
        ]);
    } else {
        echo json_encode(["status" => "error", "message" => "User not found"]);
    }
    mysqli_close($conn);
    exit();
}

// --- ২. POST এবং ৩. প্রিমিয়াম আপডেট লজিক (অপরিবর্তিত) ---
// ... (বাকি কোড আপনার ফাইলে যেমন ছিল ঠিক তেমনই থাকবে) ...

// --- ২. POST ডাটা রিসিভ করা ---
$email           = $_POST['email'] ?? '';
$action          = $_POST['action'] ?? ''; 
$package_type    = $_POST['package_type'] ?? 0;
$name            = $_POST['name'] ?? 'Unknown';
$google_id       = $_POST['google_id'] ?? '';
$photo           = $_POST['photo'] ?? ''; 
$update_mode     = $_POST['update_mode'] ?? 'false';
$update_points   = $_POST['update_points'] ?? 'false'; 
$update_tokens   = $_POST['update_tokens'] ?? 'false'; 
$update_level    = $_POST['update_level'] ?? 'false';
$amount          = $_POST['amount'] ?? 0;
$trophy_to_add   = $_POST['trophy_to_add'] ?? 0;
$new_level       = $_POST['level'] ?? 1;
$new_xp          = $_POST['xp'] ?? 0;
$perfect_wins    = $_POST['perfect_wins'] ?? 0;
$image_data      = $_POST['image_data'] ?? ''; 

if (empty($email)) {
    echo json_encode(["status" => "error", "message" => "Email is empty"]);
    exit();
}
$email = mysqli_real_escape_string($conn, $email);

// --- ৩. প্রিমিয়াম আপডেট লজিক (লাইফটাইম সাপোর্ট সহ) ---
// --- ৩. প্রিমিয়াম আপডেট লজিক (সংশোধিত) ---
if ($action === 'update_premium') {
    $days = 0;
    $cards_to_add = 0; 

    if ($package_type == 1) {
        $days = 30;
        $cards_to_add = 50; 
    } else if ($package_type == 2) {
        $days = 365;
        $cards_to_add = 600; 
    } else if ($package_type == 3) {
        $days = 18250; 
        $cards_to_add = 99999; 
    }
    
    $res = mysqli_query($conn, "SELECT room_cards FROM users WHERE email='$email'");
    $u = mysqli_fetch_assoc($res);
    
    $current_cards = (int)($u['room_cards'] ?? 0);
    $new_total_cards = $current_cards + $cards_to_add;

    // ✅ সমাধান: $base_date সবসময় NOW() ব্যবহার করুন যদি আপনি মেয়াদ আজ থেকেই শুরু করতে চান
    // অথবা আগের লজিক রাখতে চাইলে নিশ্চিত করুন যে ডাবল রিকোয়েস্ট যাচ্ছে না।
    // এখানে আমি সরাসরি আজ থেকে মেয়াদ শুরুর লজিক দিচ্ছি যাতে ডাবল না হয়:
    $base_date = "NOW()"; 

    $expiry_logic = ($package_type == 3) ? "'9999-12-31'" : "DATE_ADD($base_date, INTERVAL $days DAY)";

    $sql = "UPDATE users SET 
            premium_status = ".(int)$package_type.", 
            room_cards = $new_total_cards, 
            premium_expiry = $expiry_logic 
            WHERE email='$email'";
    
    if (mysqli_query($conn, $sql)) {
        echo json_encode([
            "status" => "success", 
            "message" => "Premium Updated", 
            "premium_status" => (int)$package_type,
            "new_room_cards" => $new_total_cards 
        ]);
    } else {
        echo json_encode(["status" => "error", "message" => mysqli_error($conn)]);
    }
    mysqli_close($conn);
    exit();
}

// --- ৪. প্রোফাইল এডিট, ৫. অন্যান্য আপডেট এবং ৬. লগইন/সাইনআপ (আপনার আগের কোড অনুযায়ী অপরিবর্তিত) ---
// ... (বাকি কোড আপনার ফাইলে যেমন ছিল ঠিক তেমনই থাকবে) ...ay php file cahnge na korle o hobe..

```

#premium_dilog.xml


```bash


<?xml version="1.0" encoding="utf-8"?>
<com.google.android.material.card.MaterialCardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:cardCornerRadius="30dp"
    app:cardElevation="20dp"
    android:layout_margin="16dp"
    app:cardBackgroundColor="#1A1040"
    app:strokeColor="#3D2A80"
    app:strokeWidth="1dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="20dp"
        android:gravity="center">

        <!-- Crown -->
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="👑"
            android:textSize="44sp"
            android:layout_marginBottom="8dp" />

        <!-- Title -->
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="GO PREMIUM"
            android:textColor="#FFFFFF"
            android:textSize="28sp"
            android:fontFamily="sans-serif-black"
            android:letterSpacing="0.08"
            android:layout_marginBottom="4dp" />

        <!-- Subtitle -->
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Unlock all features &amp; remove ads"
            android:textColor="#8B7BAA"
            android:textSize="13sp"
            android:layout_marginBottom="24dp" />

        <!-- Plan Cards Row -->
        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="bottom"
            android:weightSum="3">

            <!-- 1 Month Plan -->
            <com.google.android.material.card.MaterialCardView
                android:id="@+id/planBasic"
                android:layout_width="0dp"
                android:layout_height="160dp"
                android:layout_weight="0.9"
                app:cardCornerRadius="16dp"
                app:cardBackgroundColor="#22194E"
                app:cardElevation="4dp"
                app:strokeColor="#3D2A80"
                app:strokeWidth="1dp"
                android:layout_margin="4dp">
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="match_parent"
                    android:orientation="vertical"
                    android:gravity="top|center_horizontal">

                    <!-- Price section -->
                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="0dp"
                        android:layout_weight="1"
                        android:orientation="vertical"
                        android:gravity="center"
                        android:padding="8dp">
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="1 Month"
                            android:textSize="12sp"
                            android:textColor="#8B7BAA" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="$1.99"
                            android:textStyle="bold"
                            android:textColor="#FFFFFF"
                            android:textSize="@dimen/_12ssp" />
                    </LinearLayout>

                    <!-- Divider -->
                    <View
                        android:layout_width="match_parent"
                        android:layout_height="1dp"
                        android:background="#2A1F60" />

                    <!-- Features -->
                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="vertical"
                        android:padding="8dp"
                        android:gravity="start">

                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:layout_marginBottom="2dp"
                            android:text="• Ad-free"
                            android:textColor="#7A6A9A"
                            android:textSize="@dimen/_8ssp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• 50 room cards"
                            android:textSize="@dimen/_8ssp"
                            android:textColor="#7A6A9A"
                            android:layout_marginBottom="2dp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• Basic badge"
                            android:textSize="@dimen/_8ssp"
                            android:textColor="#7A6A9A" />
                    </LinearLayout>

                </LinearLayout>
            </com.google.android.material.card.MaterialCardView>

            <!-- 1 Year Plan (Best Value - center, taller) -->
            <com.google.android.material.card.MaterialCardView
                android:id="@+id/planPro"
                android:layout_width="0dp"
                android:layout_height="200dp"
                android:layout_weight="1.2"
                app:cardCornerRadius="20dp"
                app:cardBackgroundColor="#2A1A8C"
                app:cardElevation="12dp"
                app:strokeColor="#FFD600"
                app:strokeWidth="2dp"
                android:layout_marginHorizontal="4dp">

                <RelativeLayout
                    android:layout_width="match_parent"
                    android:layout_height="match_parent">

                    <!-- BEST VALUE badge -->
                    <TextView
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:text="BEST VALUE"
                        android:background="#FFD600"
                        android:textColor="#1A0A00"
                        android:textSize="9sp"
                        android:textStyle="bold"
                        android:textAlignment="center"
                        android:padding="5dp"
                        android:letterSpacing="0.08"
                        android:layout_alignParentTop="true" />

                    <!-- Price section -->
                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="vertical"
                        android:gravity="center"
                        android:layout_marginTop="30dp"
                        android:padding="8dp">
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="1 Year"
                            android:textColor="#C8AAFF"
                            android:textSize="13sp"
                            android:layout_marginBottom="2dp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="$9.99"
                            android:textStyle="bold"
                            android:textColor="#FFFFFF"
                            android:textSize="26sp" />

                        <!-- Save 60% pill -->
                        <com.google.android.material.card.MaterialCardView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            app:cardCornerRadius="20dp"
                            app:cardBackgroundColor="#33FFD600"
                            app:strokeColor="#FFD600"
                            app:strokeWidth="1dp"
                            app:cardElevation="0dp"
                            android:layout_marginTop="4dp">
                            <TextView
                                android:layout_width="wrap_content"
                                android:layout_height="wrap_content"
                                android:text="Save 60%"
                                android:textColor="#FFD600"
                                android:textSize="10sp"
                                android:textStyle="bold"
                                android:paddingHorizontal="10dp"
                                android:paddingVertical="3dp" />
                        </com.google.android.material.card.MaterialCardView>
                    </LinearLayout>

                    <!-- Features bottom -->
                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="vertical"
                        android:layout_alignParentBottom="true"
                        android:padding="10dp"
                        android:gravity="start">
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• Ad-free"
                            android:textSize="@dimen/_8ssp"
                            android:textColor="#C8AAFF"
                            android:layout_marginBottom="2dp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• Unlimited cards"
                            android:textSize="@dimen/_8ssp"
                            android:textColor="#C8AAFF"
                            android:layout_marginBottom="2dp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• Gold badge"
                            android:textSize="@dimen/_8ssp"
                            android:textColor="#C8AAFF"
                            android:layout_marginBottom="2dp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• Priority support"
                            android:textSize="@dimen/_8ssp"
                            android:textColor="#C8AAFF" />
                    </LinearLayout>

                </RelativeLayout>
            </com.google.android.material.card.MaterialCardView>

            <!-- Lifetime Plan -->
            <com.google.android.material.card.MaterialCardView
                android:id="@+id/planLife"
                android:layout_width="0dp"
                android:layout_height="160dp"
                android:layout_weight="0.9"
                app:cardCornerRadius="16dp"
                app:cardBackgroundColor="#22194E"
                app:cardElevation="4dp"
                app:strokeColor="#3D2A80"
                app:strokeWidth="1dp"
                android:layout_margin="4dp">
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="match_parent"
                    android:orientation="vertical"
                    android:gravity="top|center_horizontal">

                    <!-- Price section -->
                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="0dp"
                        android:layout_weight="1"
                        android:orientation="vertical"
                        android:gravity="center"
                        android:padding="8dp">
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="Lifetime"
                            android:textSize="12sp"
                            android:textColor="#8B7BAA" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="$24.99"
                            android:textStyle="bold"
                            android:textColor="#FFFFFF"
                            android:textSize="@dimen/_12ssp" />
                    </LinearLayout>

                    <!-- Divider -->
                    <View
                        android:layout_width="match_parent"
                        android:layout_height="1dp"
                        android:background="#2A1F60" />

                    <!-- Features -->
                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="vertical"
                        android:padding="8dp"
                        android:gravity="start">
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• Ad-free"
                            android:textSize="10sp"
                            android:textColor="#7A6A9A"
                            android:layout_marginBottom="2dp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• Unlimited cards"
                            android:textSize="10sp"
                            android:textColor="#7A6A9A"
                            android:layout_marginBottom="2dp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• Diamond badge"
                            android:textSize="10sp"
                            android:textColor="#7A6A9A"
                            android:layout_marginBottom="2dp" />
                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="• All future features"
                            android:textSize="10sp"
                            android:textColor="#7A6A9A" />
                    </LinearLayout>

                </LinearLayout>
            </com.google.android.material.card.MaterialCardView>

        </LinearLayout>

        <!-- Divider line -->
        <View
            android:layout_width="match_parent"
            android:layout_height="1dp"
            android:background="#2A1F60"
            android:layout_marginTop="20dp"
            android:layout_marginBottom="16dp" />

        <!-- Continue Button -->
        <com.google.android.material.button.MaterialButton
            android:id="@+id/btnSubscribe"
            android:layout_width="match_parent"
            android:layout_height="60dp"
            android:text="CONTINUE"
            android:textSize="16sp"
            android:textStyle="bold"
            android:letterSpacing="0.08"
            app:cornerRadius="15dp"
            app:backgroundTint="#22194E"
            app:strokeColor="#4A3590"
            app:strokeWidth="1dp" />

        <!-- Maybe Later -->
        <TextView
            android:id="@+id/tvClose"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Maybe Later"
            android:layout_marginTop="16dp"
            android:textColor="#4A3A6A"
            android:textSize="13sp"
            android:padding="8dp" />

    </LinearLayout>
</com.google.android.material.card.MaterialCardView>

```

#shop_card_dilog.xml

```bash

<?xml version="1.0" encoding="utf-8"?>
<com.google.android.material.card.MaterialCardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    app:cardCornerRadius="28dp"
    app:cardElevation="12dp"
    android:layout_margin="16dp"
    app:cardBackgroundColor="#FFFFFF">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="24dp">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="CARD SHOP"
            android:textAlignment="center"
            android:textColor="#1A237E"
            android:textSize="24sp"
            android:fontFamily="sans-serif-black"
            android:letterSpacing="0.1" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Boost your battle power with premium cards"
            android:textAlignment="center"
            android:textColor="#757575"
            android:textSize="12sp"
            android:layout_marginBottom="24dp" />

        <com.google.android.material.card.MaterialCardView
            android:id="@+id/btnDailyClaim"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:cardCornerRadius="16dp"
            app:cardBackgroundColor="#E8F5E9"
            app:cardElevation="0dp"
            android:clickable="true"
            android:focusable="true"
            android:layout_marginBottom="12dp">

            <RelativeLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:padding="16dp">
                <ImageView
                    android:id="@+id/iconGift"
                    android:layout_width="40dp"
                    android:layout_height="40dp"
                    android:src="@drawable/room_card" />
                <LinearLayout
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:layout_toEndOf="@id/iconGift"
                    android:layout_marginStart="16dp">
                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Daily Reward"
                        android:textColor="#2E7D32"
                        android:textSize="16sp"
                        android:textStyle="bold" />
                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Claim 1 free card every 24h"
                        android:textSize="11sp"
                        android:textColor="#43A047" />
                </LinearLayout>
                <com.google.android.material.button.MaterialButton
                    android:id="@+id/tvClaimAction"
                    style="@style/Widget.MaterialComponents.Button.TextButton"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:layout_alignParentEnd="true"
                    android:layout_centerVertical="true"
                    android:text="CLAIM"
                    android:textColor="#2E7D32"
                    android:textStyle="bold" />
            </RelativeLayout>
        </com.google.android.material.card.MaterialCardView>

        <com.google.android.material.card.MaterialCardView
            android:id="@+id/btnWatchAds"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:cardCornerRadius="16dp"
            app:cardBackgroundColor="#FFF3E0"
            app:cardElevation="0dp"
            android:clickable="true"
            android:focusable="true"
            android:layout_marginBottom="24dp">

            <RelativeLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:padding="16dp">
                <TextView
                    android:id="@+id/iconAds"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:text="📺"
                    android:textSize="28sp" />
                <LinearLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:orientation="vertical"
                    android:layout_toEndOf="@id/iconAds"
                    android:layout_toStartOf="@id/tvWatchBtnText"
                    android:layout_marginStart="16dp"
                    android:layout_marginEnd="8dp">
                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Watch &amp; Earn"
                        android:textColor="#E65100"
                        android:textSize="16sp"
                        android:textStyle="bold" />
                    <TextView
                        android:id="@+id/tvAdStatus"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="Watched 0/3 ads"
                        android:textSize="11sp"
                        android:textColor="#EF6C00"
                        android:layout_marginTop="2dp" />
                    <ProgressBar
                        android:id="@+id/adProgressBar"
                        style="?android:attr/progressBarStyleHorizontal"
                        android:layout_width="match_parent"
                        android:layout_height="8dp"
                        android:max="3"
                        android:progress="1"
                        android:progressTint="#FB8C00"
                        android:layout_marginTop="8dp"/>
                </LinearLayout>
                <TextView
                    android:id="@+id/tvWatchBtnText"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:layout_alignParentEnd="true"
                    android:layout_centerVertical="true"
                    android:text="WATCH"
                    android:textColor="#E65100"
                    android:textStyle="bold" />
            </RelativeLayout>
        </com.google.android.material.card.MaterialCardView>

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="BEST VALUE OFFERS"
            android:textColor="#BDBDBD"
            android:textSize="11sp"
            android:letterSpacing="0.1"
            android:textStyle="bold"
            android:layout_marginBottom="16dp" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:visibility="gone">

            <com.google.android.material.card.MaterialCardView
                android:id="@+id/buy5"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                app:cardCornerRadius="20dp"
                app:cardBackgroundColor="#F8F9FA"
                app:cardElevation="2dp"
                android:layout_marginEnd="8dp">

                <RelativeLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:padding="12dp">

                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="5X"
                        android:background="#1A237E"
                        android:textColor="#FFFFFF"
                        android:textSize="10sp"
                        android:textStyle="bold"
                        android:paddingHorizontal="8dp"
                        android:paddingVertical="2dp" />

                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:layout_marginTop="12dp"
                        android:gravity="center"
                        android:orientation="vertical">

                        <ImageView
                            android:layout_width="80dp"
                            android:layout_height="60dp"
                            android:src="@drawable/room_card" />

                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:layout_marginTop="8dp"
                            android:text="Starter Pack"
                            android:textColor="#616161"
                            android:textSize="12sp" />

                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="$0.99"
                            android:textColor="#212121"
                            android:textSize="18sp"
                            android:textStyle="bold" />
                    </LinearLayout>
                </RelativeLayout>
            </com.google.android.material.card.MaterialCardView>

            <com.google.android.material.card.MaterialCardView
                android:id="@+id/buy10"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                app:cardCornerRadius="20dp"
                app:cardBackgroundColor="#FFFDE7"
                app:cardElevation="4dp"
                app:strokeColor="#FFD600"
                app:strokeWidth="1dp"
                android:layout_marginStart="8dp">

                <RelativeLayout
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:padding="12dp">

                    <TextView
                        android:id="@+id/badge10x"
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:text="10X"
                        android:background="#D50000"
                        android:textColor="#FFFFFF"
                        android:textSize="10sp"
                        android:textStyle="bold"
                        android:paddingHorizontal="8dp"
                        android:paddingVertical="2dp" />

                    <TextView
                        android:layout_width="wrap_content"
                        android:layout_height="wrap_content"
                        android:layout_alignParentEnd="true"
                        android:text="-20%"
                        android:textColor="#D50000"
                        android:textSize="10sp"
                        android:textStyle="bold" />

                    <LinearLayout
                        android:layout_width="match_parent"
                        android:layout_height="wrap_content"
                        android:orientation="vertical"
                        android:gravity="center"
                        android:layout_marginTop="12dp">

                        <ImageView
                            android:layout_width="80dp"
                            android:layout_height="60dp"
                            android:src="@drawable/room_card" />

                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="Pro Pack"
                            android:textSize="12sp"
                            android:textColor="#616161"
                            android:layout_marginTop="8dp"/>

                        <TextView
                            android:layout_width="wrap_content"
                            android:layout_height="wrap_content"
                            android:text="$1.99"
                            android:textColor="#000000"
                            android:textSize="18sp"
                            android:textStyle="bold" />
                    </LinearLayout>
                </RelativeLayout>
            </com.google.android.material.card.MaterialCardView>

        </LinearLayout>

        <TextView
            android:id="@+id/btnClose"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Close Shop"
            android:textAlignment="center"
            android:layout_marginTop="24dp"
            android:textColor="#9E9E9E"
            android:textSize="14sp"
            android:padding="8dp" />

    </LinearLayout>
</com.google.android.material.card.MaterialCardView>


```








