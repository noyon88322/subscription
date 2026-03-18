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



```bash







````

