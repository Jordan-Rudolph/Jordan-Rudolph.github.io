---
layout: default
title: Grub Bug Code Samples
description: ⠀
---

[back to main page](./)

# AudioManager (C# and Unity engine)

```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
using UnityEngine.SceneManagement;
using UnityEngine.Assertions;

/// <summary>
/// Centralized audio management system for Unity games.
/// Handles music playback, sound effects, spatial audio, and volume controls.
/// Implements singleton pattern and persists across scene changes.
/// </summary>
public class AudioManager : MonoBehaviour
{
    #region Singleton Implementation
    
    private static AudioManager instance;
    
    /// <summary>
    /// Global access point for the AudioManager instance.
    /// Creates a new instance if none exists.
    /// </summary>
    public static AudioManager Instance
    {
        get
        {
            if (instance == null)
            {
                instance = FindObjectOfType<AudioManager>();
                if (instance == null)
                {
                    GameObject audioManagerObject = new GameObject("AudioManager");
                    instance = audioManagerObject.AddComponent<AudioManager>();
                }
            }
            return instance;
        }
    }
    
    #endregion

    #region Public Fields
    
    [Header("Audio Configuration")]
    [SerializeField] private Sound[] musicTracks;
    [SerializeField] private Sound[] soundEffects;
    
    #endregion

    #region Private Fields
    
    private Sound currentMusic;
    private Dictionary<string, Sound> soundDictionary = new Dictionary<string, Sound>();
    private List<AudioSource> temporaryAudioSources = new List<AudioSource>();
    
    // Audio configuration constants
    private const float LOWEST_PITCH = 0.5f;
    private const float PEDESTRIAN_VOLUME_BOOST = 0.4f;
    private const float PEDESTRIAN_HURT_VOLUME_BOOST = 0.15f;
    private const float SPATIAL_MIN_DISTANCE = 40f;
    private const float SPATIAL_MAX_DISTANCE = 500f;
    
    private float globalPitch = 1f;
    
    #endregion

    #region Unity Lifecycle
    
    /// <summary>
    /// Initialize the AudioManager singleton and set up audio sources.
    /// </summary>
    private void Awake()
    {
        // Enforce singleton pattern
        if (instance != null && instance != this)
        {
            Destroy(gameObject);
            return;
        }

        instance = this;
        DontDestroyOnLoad(gameObject);
        
        // Subscribe to scene change events for cleanup
        SceneManager.sceneLoaded += OnSceneLoaded;

        InitializeAudioSources();
    }
    
    /// <summary>
    /// Clean up when the AudioManager is destroyed.
    /// </summary>
    private void OnDestroy()
    {
        SceneManager.sceneLoaded -= OnSceneLoaded;
    }
    
    #endregion

    #region Initialization
    
    /// <summary>
    /// Initialize all audio sources for music tracks and sound effects.
    /// Sets up the sound dictionary for quick access.
    /// </summary>
    private void InitializeAudioSources()
    {
        InitializeMusicTracks();
        InitializeSoundEffects();
        ApplyReverbEffects();
    }
    
    /// <summary>
    /// Set up audio sources for all music tracks.
    /// </summary>
    private void InitializeMusicTracks()
    {
        foreach (Sound musicTrack in musicTracks)
        {
            ConfigureAudioSource(musicTrack, isMusic: true);
            soundDictionary[musicTrack.name] = musicTrack;
        }
    }
    
    /// <summary>
    /// Set up audio sources for all sound effects with special handling for certain types.
    /// </summary>
    private void InitializeSoundEffects()
    {
        foreach (Sound soundEffect in soundEffects)
        {
            CreateAudioSourceForSoundEffect(soundEffect);
            ConfigureAudioSource(soundEffect, isMusic: false);
            ApplySpecialVolumeSettings(soundEffect);
            soundDictionary[soundEffect.name] = soundEffect;
        }
    }
    
    /// <summary>
    /// Create appropriate audio source container for sound effects.
    /// Uses child objects for sounds requiring reverb.
    /// </summary>
    private void CreateAudioSourceForSoundEffect(Sound soundEffect)
    {
        if (soundEffect.hasReverb())
        {
            GameObject reverbContainer = new GameObject($"{soundEffect.name}_ReverbContainer");
            reverbContainer.transform.SetParent(transform);
            soundEffect.source = reverbContainer.AddComponent<AudioSource>();
        }
        else
        {
            soundEffect.source = gameObject.AddComponent<AudioSource>();
        }
    }
    
    /// <summary>
    /// Configure basic audio source properties.
    /// </summary>
    private void ConfigureAudioSource(Sound sound, bool isMusic)
    {
        sound.source.clip = sound.clip;
        sound.source.pitch = sound.pitch;
        sound.source.loop = sound.loop;
        sound.setMusic(isMusic);
    }
    
    /// <summary>
    /// Apply special volume settings for pedestrian character sounds.
    /// </summary>
    private void ApplySpecialVolumeSettings(Sound soundEffect)
    {
        if (IsPedestrianSound(soundEffect.name))
        {
            float volumeBoost = PEDESTRIAN_VOLUME_BOOST;
            if (soundEffect.name.ContainsInsensitive("hurt"))
            {
                volumeBoost += PEDESTRIAN_HURT_VOLUME_BOOST;
            }
            soundEffect.source.volume = soundEffect.volume + volumeBoost;
        }
        else
        {
            soundEffect.source.volume = soundEffect.volume;
        }
    }
    
    /// <summary>
    /// Check if a sound belongs to pedestrian characters.
    /// </summary>
    private bool IsPedestrianSound(string soundName)
    {
        return soundName.ContainsInsensitive("Beetle") ||
               soundName.ContainsInsensitive("Grasshopper") ||
               soundName.ContainsInsensitive("Ladybug") ||
               soundName.ContainsInsensitive("Rolypoly");
    }
    
    /// <summary>
    /// Apply reverb effects to sounds that require them.
    /// </summary>
    private void ApplyReverbEffects()
    {
        foreach (Sound soundEffect in soundEffects)
        {
            if (soundEffect.hasReverb())
            {
                soundDictionary[soundEffect.name].source.gameObject.AddComponent<AudioReverbFilter>();
            }
        }
    }
    
    #endregion

    #region Music Control
    
    /// <summary>
    /// Play a music track with optional fade-in effect.
    /// Automatically fades out currently playing music.
    /// </summary>
    /// <param name="trackName">Name of the music track to play</param>
    /// <param name="fadeInDuration">Duration of fade-in effect in seconds</param>
    public void PlayMusic(string trackName, float fadeInDuration = 0f)
    {
        if (!soundDictionary.TryGetValue(trackName, out Sound newMusic))
        {
            Debug.LogWarning($"Music track '{trackName}' not found in audio dictionary!");
            return;
        }

        // Fade out current music if playing
        if (currentMusic != null && currentMusic.source.isPlaying)
        {
            StartCoroutine(FadeOut(currentMusic.source, fadeInDuration));
        }

        // Start new music with fade-in
        StartCoroutine(FadeIn(newMusic.source, newMusic.volume, fadeInDuration));
        currentMusic = newMusic;
    }
    
    /// <summary>
    /// Stop currently playing music with optional fade-out effect.
    /// </summary>
    /// <param name="fadeOutDuration">Duration of fade-out effect in seconds</param>
    public void StopMusic(float fadeOutDuration = 0f)
    {
        if (currentMusic != null && currentMusic.source.isPlaying)
        {
            StartCoroutine(FadeOut(currentMusic.source, fadeOutDuration));
            currentMusic = null;
        }
    }
    
    #endregion

    #region Sound Effect Control
    
    /// <summary>
    /// Play a sound effect by name.
    /// </summary>
    /// <param name="soundName">Name of the sound effect to play</param>
    public void PlaySound(string soundName)
    {
        if (!soundDictionary.TryGetValue(soundName, out Sound sound))
        {
            Debug.LogWarning($"Sound effect '{soundName}' not found in audio dictionary!");
            return;
        }
        
        sound.source.Play();
    }
    
    /// <summary>
    /// Play a sound effect at a specific world position with 3D spatial audio.
    /// Creates a temporary audio source that self-destructs after playback.
    /// </summary>
    /// <param name="soundName">Name of the sound effect to play</param>
    /// <param name="position">World position where the sound should originate</param>
    public void PlaySoundAtPoint(string soundName, Vector3 position)
    {
        if (!soundDictionary.TryGetValue(soundName, out Sound sound))
        {
            Debug.LogWarning($"Sound effect '{soundName}' not found for spatial playback!");
            return;
        }

        CleanupFinishedTemporaryAudioSources();
        
        // Create temporary audio source for spatial playback
        GameObject spatialAudioObject = new GameObject($"SpatialAudio_{soundName}");
        spatialAudioObject.transform.position = position;
        
        AudioSource temporarySource = spatialAudioObject.AddComponent<AudioSource>();
        ConfigureSpatialAudioSource(temporarySource, sound);
        
        temporaryAudioSources.Add(temporarySource);
        temporarySource.Play();
        
        // Schedule cleanup after sound finishes
        float destructionDelay = sound.source.clip.length * (1.0f / LOWEST_PITCH);
        Destroy(spatialAudioObject, destructionDelay);
    }
    
    /// <summary>
    /// Stop a specific sound effect.
    /// </summary>
    /// <param name="soundName">Name of the sound effect to stop</param>
    public void StopSound(string soundName)
    {
        if (!soundDictionary.TryGetValue(soundName, out Sound sound))
        {
            Debug.LogWarning($"Sound effect '{soundName}' not found for stopping!");
            return;
        }
        
        sound.source.Stop();
    }
    
    /// <summary>
    /// Check if a specific sound is currently playing.
    /// </summary>
    /// <param name="soundName">Name of the sound to check</param>
    /// <returns>True if the sound is playing, false otherwise</returns>
    public bool IsSoundPlaying(string soundName)
    {
        if (!soundDictionary.TryGetValue(soundName, out Sound sound))
        {
            Debug.LogWarning($"Sound effect '{soundName}' not found for status check!");
            return false;
        }
        
        return sound.source.isPlaying;
    }
    
    /// <summary>
    /// Set whether a sound should loop.
    /// </summary>
    /// <param name="soundName">Name of the sound to modify</param>
    /// <param name="shouldLoop">Whether the sound should loop</param>
    public void SetLooping(string soundName, bool shouldLoop)
    {
        if (!soundDictionary.TryGetValue(soundName, out Sound sound))
        {
            Debug.LogWarning($"Sound effect '{soundName}' not found for loop setting!");
            return;
        }

        sound.source.loop = shouldLoop;
    }
    
    #endregion

    #region Audio Effects & Processing
    
    /// <summary>
    /// Gradually change the pitch of all audio over a specified duration.
    /// Useful for creating slow-motion or time-acceleration effects.
    /// </summary>
    /// <param name="targetPitch">Target pitch value (minimum: 0.5)</param>
    /// <param name="duration">Duration of the pitch change in seconds</param>
    public IEnumerator ChangePitch(float targetPitch, float duration)
    {
        Assert.IsTrue(targetPitch >= LOWEST_PITCH, $"Pitch must be at least {LOWEST_PITCH}");
        
        float startTime = Time.time;
        float startPitch = globalPitch;

        while (Time.time < startTime + duration)
        {
            float elapsed = Time.time - startTime;
            float normalizedTime = elapsed / duration;
            float currentPitch = Mathf.Lerp(startPitch, targetPitch, normalizedTime);
            
            ApplyPitchToAllAudio(currentPitch);
            yield return null;
        }

        // Ensure final pitch is exactly the target
        ApplyPitchToAllAudio(targetPitch);
    }
    
    /// <summary>
    /// Apply pitch modification to all active audio sources.
    /// </summary>
    private void ApplyPitchToAllAudio(float pitch)
    {
        globalPitch = pitch;
        
        // Apply to all registered sounds
        foreach (KeyValuePair<string, Sound> soundPair in soundDictionary)
        {
            soundPair.Value.source.pitch = pitch;
        }
        
        // Apply to temporary spatial audio sources
        foreach (AudioSource temporarySource in temporaryAudioSources)
        {
            if (temporarySource != null)
            {
                temporarySource.pitch = pitch;
            }
        }
    }
    
    #endregion

    #region Volume Management
    
    /// <summary>
    /// Update all audio source volumes based on current PlayerPrefs settings.
    /// Call this when volume settings change in the options menu.
    /// </summary>
    public void LoadVolumeSettings()
    {
        float masterVolume = PlayerPrefs.GetFloat("MasterVolumeKey", 1f);
        float musicVolume = PlayerPrefs.GetFloat("MusicKey", 1f);
        float sfxVolume = PlayerPrefs.GetFloat("SFXKey", 1f);
        
        foreach (KeyValuePair<string, Sound> soundPair in soundDictionary)
        {
            Sound sound = soundPair.Value;
            float volumeMultiplier = sound.isMusic() ? musicVolume : sfxVolume;
            sound.source.volume = masterVolume * volumeMultiplier * sound.volume;
        }
    }
    
    #endregion

    #region Audio Fade Effects
    
    /// <summary>
    /// Fade in an audio source over a specified duration.
    /// </summary>
    private System.Collections.IEnumerator FadeIn(AudioSource audioSource, float targetVolume, float duration)
    {
        float masterVolume = PlayerPrefs.GetFloat("MasterVolumeKey", 1f);
        float musicVolume = PlayerPrefs.GetFloat("MusicKey", 1f);
        float finalVolume = masterVolume * musicVolume * targetVolume;
        
        audioSource.volume = 0f;
        audioSource.Play();

        if (duration <= 0f)
        {
            audioSource.volume = finalVolume;
            yield break;
        }

        float startTime = Time.time;
        while (Time.time < startTime + duration)
        {
            float elapsed = Time.time - startTime;
            float normalizedTime = elapsed / duration;
            audioSource.volume = Mathf.Lerp(0f, finalVolume, normalizedTime);
            yield return null;
        }

        audioSource.volume = finalVolume;
    }

    /// <summary>
    /// Fade out an audio source over a specified duration.
    /// </summary>
    private System.Collections.IEnumerator FadeOut(AudioSource audioSource, float duration)
    {
        if (duration <= 0f)
        {
            audioSource.Stop();
            yield break;
        }

        float startVolume = audioSource.volume;
        float startTime = Time.time;

        while (Time.time < startTime + duration)
        {
            float elapsed = Time.time - startTime;
            float normalizedTime = elapsed / duration;
            audioSource.volume = Mathf.Lerp(startVolume, 0f, normalizedTime);
            yield return null;
        }

        audioSource.Stop();
        audioSource.volume = startVolume; // Reset for potential reuse
    }
    
    #endregion

    #region Spatial Audio Configuration
    
    /// <summary>
    /// Configure an audio source for 3D spatial playback.
    /// </summary>
    private void ConfigureSpatialAudioSource(AudioSource audioSource, Sound referenceSound)
    {
        // Copy properties from reference sound
        audioSource.clip = referenceSound.source.clip;
        audioSource.volume = referenceSound.source.volume;
        
        // Configure 3D spatial audio settings
        audioSource.dopplerLevel = 0f;
        audioSource.spatialBlend = 1f; // Full 3D positioning
        audioSource.minDistance = SPATIAL_MIN_DISTANCE;
        audioSource.maxDistance = SPATIAL_MAX_DISTANCE;
    }
    
    #endregion

    #region Cleanup & Maintenance
    
    /// <summary>
    /// Remove finished temporary audio sources to prevent memory leaks.
    /// </summary>
    private void CleanupFinishedTemporaryAudioSources()
    {
        for (int i = temporaryAudioSources.Count - 1; i >= 0; i--)
        {
            AudioSource source = temporaryAudioSources[i];
            if (source == null || !source.isPlaying)
            {
                temporaryAudioSources.RemoveAt(i);
            }
        }
    }
    
    /// <summary>
    /// Handle scene loading events. Stop specific sounds that shouldn't persist.
    /// </summary>
    private void OnSceneLoaded(Scene scene, LoadSceneMode loadMode)
    {
        // Stop scene-specific sounds
        StopSound("sfx_SirenLong");
        
        // Clean up any remaining temporary audio sources
        CleanupFinishedTemporaryAudioSources();
    }
    
    #endregion
}
```

[back to main page](./)