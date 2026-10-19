
import React, { useEffect, useState } from 'react';
import {
  View,
  Text,
  FlatList,
  Image,
  TextInput,
  TouchableOpacity,
  ActivityIndicator,
  StyleSheet,
  StatusBar,
  SafeAreaView,
  ScrollView,
} from 'react-native';

const API = 'https://rickandmortyapi.com/api';

const TABS = [
  { key: 'character', title: 'Персонажи', icon: '👽' },
  { key: 'location', title: 'Локации', icon: '🌎' },
  { key: 'episode', title: 'Эпизоды', icon: '📺' },
];

const COLORS = {
  background: '#0b1017',
  card: '#151e29',
  green: '#b7f34a',
  text: '#f4f7fb',
  muted: '#91a0b2',
  border: '#273444',
  red: '#ff6565',
};

function getStatusColor(status) {
  if (status === 'Alive') return COLORS.green;
  if (status === 'Dead') return COLORS.red;
  return COLORS.muted;
}

function getSubtitle(item, type) {
  if (type === 'character') {
    return `${item.species} · ${item.gender}`;
  }
  if (type === 'location') {
    return `${item.type} · ${item.dimension}`;
  }
  return `${item.episode} · ${item.air_date}`;
}

export default function App() {
  const [activeTab, setActiveTab] = useState('character');
  const [items, setItems] = useState([]);
  const [selected, setSelected] = useState(null);
  const [searchInput, setSearchInput] = useState('');
  const [appliedSearch, setAppliedSearch] = useState('');
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(false);
  const [loading, setLoading] = useState(true);
  const [loadingMore, setLoadingMore] = useState(false);
  const [error, setError] = useState('');
  const [retryKey, setRetryKey] = useState(0);

  async function loadData(type, query, pageNumber, append = false) {
    if (append) {
      setLoadingMore(true);
    } else {
      setLoading(true);
      setError('');
    }

    try {
      let url = `${API}/${type}?page=${pageNumber}`;

      if (query.trim()) {
        url += `&name=${encodeURIComponent(query.trim())}`;
      }

      const response = await fetch(url);
      const data = await response.json();

      if (!response.ok) {
        throw new Error(
          data.error || `Ошибка сервера: ${response.status}`
        );
      }

      setItems(previous =>
        append ? [...previous, ...data.results] : data.results
      );
      setPage(pageNumber);
      setHasMore(Boolean(data.info.next));
    } catch (e) {
      if (!append) {
        setItems([]);
        setHasMore(false);
        setError(
          e.message === 'Network request failed'
            ? 'Нет соединения с интернетом. Проверь подключение.'
            : e.message || 'Не удалось загрузить данные.'
        );
      }
    } finally {
      setLoading(false);
      setLoadingMore(false);
    }
  }

  useEffect(() => {
    loadData(activeTab, appliedSearch, 1);
  }, [activeTab, appliedSearch, retryKey]);

  function changeTab(key) {
    setSelected(null);
    setSearchInput('');
    setAppliedSearch('');
    setActiveTab(key);
  }

  function search() {
    setSelected(null);
    setAppliedSearch(searchInput.trim());
  }

  function clearSearch() {
    setSearchInput('');
    setAppliedSearch('');
  }

  function loadNextPage() {
    if (!loading && !loadingMore && hasMore && !error) {
      loadData(activeTab, appliedSearch, page + 1, true);
    }
  }

  function renderCard({ item }) {
    const isCharacter = activeTab === 'character';

    return (
      <TouchableOpacity
        style={styles.card}
        activeOpacity={0.8}
        onPress={() => setSelected(item)}
      >
        {isCharacter ? (
          <Image
            source={{ uri: item.image }}
            style={styles.avatar}
          />
        ) : (
          <View style={styles.iconBox}>
            <Text style={styles.bigIcon}>
              {activeTab === 'location' ? '🌌' : '🎬'}
            </Text>
          </View>
        )}

        <View style={styles.cardInfo}>
          <Text style={styles.cardTitle} numberOfLines={2}>
            {item.name}
          </Text>

          {isCharacter && (
            <View style={styles.statusRow}>
              <View
                style={[
                  styles.statusDot,
                  { backgroundColor: getStatusColor(item.status) },
                ]}
              />
              <Text style={styles.mutedText}>
                {item.status === 'Alive'
                  ? 'Жив'
                  : item.status === 'Dead'
                  ? 'Мёртв'
                  : 'Неизвестно'}
              </Text>
            </View>
          )}

          <Text style={styles.subtitle} numberOfLines={2}>
            {getSubtitle(item, activeTab)}
          </Text>

          <Text style={styles.moreText}>Подробнее →</Text>
        </View>
      </TouchableOpacity>
    );
  }

  function renderDetail() {
    const item = selected;
    if (!item) return null;

    const isCharacter = activeTab === 'character';
    const isLocation = activeTab === 'location';

    const details = isCharacter
      ? [
          ['Статус', item.status],
          ['Вид', item.species],
          ['Тип', item.type || 'Не указан'],
          ['Пол', item.gender],
          ['Происхождение', item.origin?.name],
          ['Последнее местоположение', item.location?.name],
          ['Количество эпизодов', item.episode?.length],
          ['ID', item.id],
        ]
      : isLocation
      ? [
          ['Тип', item.type],
          ['Измерение', item.dimension],
          ['Количество жителей', item.residents?.length],
          ['ID', item.id],
          ['Создано', item.created?.slice(0, 10)],
        ]
      : [
          ['Код эпизода', item.episode],
          ['Дата выхода', item.air_date],
          ['Количество персонажей', item.characters?.length],
          ['ID', item.id],
          ['Создано', item.created?.slice(0, 10)],
        ];

    return (
      <View style={styles.screen}>
        <TouchableOpacity
          style={styles.backButton}
          onPress={() => setSelected(null)}
        >
          <Text style={styles.backText}>← Назад к списку</Text>
        </TouchableOpacity>

        <ScrollView showsVerticalScrollIndicator={false}>
          {isCharacter ? (
            <Image
              source={{ uri: item.image }}
              style={styles.detailAvatar}
            />
          ) : (
            <View style={styles.detailIcon}>
              <Text style={styles.detailEmoji}>
                {isLocation ? '🌌' : '🎬'}
              </Text>
            </View>
          )}

          <Text style={styles.detailTitle}>{item.name}</Text>
          <Text style={styles.detailSubtitle}>
            {getSubtitle(item, activeTab)}
          </Text>

          <Text style={styles.sectionTitle}>Информация</Text>

          {details.map(([label, value]) => (
            <View key={label} style={styles.detailRow}>
              <Text style={styles.detailLabel}>{label}</Text>
              <Text style={styles.detailValue}>
                {String(value ?? 'Не указано')}
              </Text>
            </View>
          ))}

          <Text style={styles.footerNote}>
            Данные получены из Rick and Morty API
          </Text>
        </ScrollView>
      </View>
    );
  }

  return (
    <SafeAreaView style={styles.container}>
      <StatusBar barStyle="light-content" backgroundColor={COLORS.background} />

      {selected ? (
        renderDetail()
      ) : (
        <>
          <View style={styles.header}>
            <Text style={styles.brand}>RICK<Text style={styles.brandGreen}> & </Text>MORTY</Text>
            <Text style={styles.headerSubtitle}>UNIVERSE DATABASE</Text>
          </View>

          <View style={styles.tabs}>
            {TABS.map(tab => {
              const active = activeTab === tab.key;

              return (
                <TouchableOpacity
                  key={tab.key}
                  onPress={() => changeTab(tab.key)}
                  style={[styles.tab, active && styles.activeTab]}
                >
                  <Text style={styles.tabIcon}>{tab.icon}</Text>
                  <Text
                    style={[styles.tabText, active && styles.activeTabText]}
                  >
                    {tab.title}
                  </Text>
                </TouchableOpacity>
              );
            })}
          </View>

          <View style={styles.searchRow}>
            <TextInput
              style={styles.searchInput}
              value={searchInput}
              onChangeText={setSearchInput}
              placeholder={`Поиск: ${TABS.find(t => t.key === activeTab)?.title.toLowerCase()}...`}
              placeholderTextColor={COLORS.muted}
              returnKeyType="search"
              onSubmitEditing={search}
            />
            <TouchableOpacity style={styles.searchButton} onPress={search}>
              <Text style={styles.searchButtonText}>Найти</Text>
            </TouchableOpacity>
          </View>

          {appliedSearch !== '' && (
            <View style={styles.filterRow}>
              <Text style={styles.mutedText}>
                Результаты: «{appliedSearch}»
              </Text>
              <TouchableOpacity onPress={clearSearch}>
                <Text style={styles.clearText}>Сбросить ✕</Text>
              </TouchableOpacity>
            </View>
          )}

          {!loading && !error && (
            <Text style={styles.resultCount}>
              Найдено: {items.length}{hasMore ? '+' : ''}
            </Text>
          )}

          {loading ? (
            <View style={styles.center}>
              <ActivityIndicator size="large" color={COLORS.green} />
              <Text style={styles.loadingText}>Загружаем данные...</Text>
            </View>
          ) : error ? (
            <View style={styles.center}>
              <Text style={styles.errorIcon}>⚠️</Text>
              <Text style={styles.errorTitle}>Что-то пошло не так</Text>
              <Text style={styles.errorText}>{error}</Text>
              <TouchableOpacity
                style={styles.retryButton}
                onPress={() => setRetryKey(k => k + 1)}
              >
                <Text style={styles.retryText}>Повторить загрузку</Text>
              </TouchableOpacity>
            </View>
          ) : (
            <FlatList
              key={activeTab}
              data={items}
              renderItem={renderCard}
              keyExtractor={item => String(item.id)}
              contentContainerStyle={styles.list}
              showsVerticalScrollIndicator={false}
              onEndReached={loadNextPage}
              onEndReachedThreshold={0.4}
              ListEmptyComponent={
                <Text style={styles.emptyText}>
                  Ничего не найдено. Попробуй другой запрос.
                </Text>
              }
              ListFooterComponent={
                loadingMore ? (
                  <ActivityIndicator
                    style={{ margin: 20 }}
                    color={COLORS.green}
                    size="large"
                  />
                ) : hasMore ? (
                  <TouchableOpacity
                    style={styles.loadMoreButton}
                    onPress={loadNextPage}
                  >
                    <Text style={styles.loadMoreText}>
                      Загрузить ещё
                    </Text>
                  </TouchableOpacity>
                ) : items.length > 0 ? (
                  <Text style={styles.endText}>Все записи загружены</Text>
                ) : null
              }
            />
          )}

          <View style={styles.bottomBar}>
            <Text style={styles.bottomText}>
              POWERED BY RICKANDMORTYAPI.COM
            </Text>
          </View>
        </>
      )}
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: COLORS.background,
  },
  screen: {
    flex: 1,
    paddingHorizontal: 18,
    paddingTop: 12,
  },
  header: {
    paddingHorizontal: 20,
    paddingTop: 20,
    paddingBottom: 18,
  },
  brand: {
    color: COLORS.text,
    fontSize: 27,
    fontWeight: '900',
    letterSpacing: 1.5,
  },
  brandGreen: {
    color: COLORS.green,
  },
  headerSubtitle: {
    color: COLORS.muted,
    fontSize: 10,
    letterSpacing: 3,
    marginTop: 5,
  },
  tabs: {
    flexDirection: 'row',
    marginHorizontal: 14,
    marginBottom: 14,
    backgroundColor: COLORS.card,
    borderRadius: 14,
    padding: 5,
  },
  tab: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    paddingVertical: 11,
    borderRadius: 10,
  },
  activeTab: {
    backgroundColor: COLORS.green,
  },
  tabIcon: {
    fontSize: 18,
    marginBottom: 4,
  },
  tabText: {
    fontSize: 11,
    color: COLORS.muted,
    fontWeight: '600',
  },
  activeTabText: {
    color: '#11170b',
  },
  searchRow: {
    flexDirection: 'row',
    marginHorizontal: 14,
    gap: 8,
    marginBottom: 10,
  },
  searchInput: {
    flex: 1,
    backgroundColor: COLORS.card,
    borderWidth: 1,
    borderColor: COLORS.border,
    borderRadius: 12,
    paddingHorizontal: 14,
    color: COLORS.text,
    height: 46,
  },
  searchButton: {
    backgroundColor: COLORS.green,
    paddingHorizontal: 16,
    justifyContent: 'center',
    borderRadius: 12,
  },
  searchButtonText: {
    color: '#11170b',
    fontWeight: '800',
  },
  filterRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    paddingHorizontal: 20,
    paddingBottom: 8,
  },
  clearText: {
    color: COLORS.green,
    fontSize: 12,
  },
  resultCount: {
    color: COLORS.muted,
    fontSize: 12,
    paddingHorizontal: 20,
    paddingBottom: 8,
  },
  list: {
    paddingHorizontal: 14,
    paddingBottom: 20,
  },
  card: {
    flexDirection: 'row',
    backgroundColor: COLORS.card,
    borderWidth: 1,
    borderColor: COLORS.border,
    borderRadius: 15,
    padding: 11,
    marginBottom: 10,
    alignItems: 'center',
  },
  avatar: {
    width: 100,
    height: 100,
    borderRadius: 12,
    backgroundColor: '#263443',
  },
  iconBox: {
    width: 100,
    height: 100,
    borderRadius: 12,
    backgroundColor: '#263443',
    justifyContent: 'center',
    alignItems: 'center',
  },
  bigIcon: {
    fontSize: 38,
  },
  cardInfo: {
    flex: 1,
    marginLeft: 13,
  },
  cardTitle: {
    color: COLORS.text,
    fontSize: 16,
    fontWeight: '800',
    marginBottom: 7,
  },
  statusRow: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 6,
  },
  statusDot: {
    width: 7,
    height: 7,
    borderRadius: 4,
    marginRight: 7,
  },
  mutedText: {
    color: COLORS.muted,
    fontSize: 12,
  },
  subtitle: {
    color: COLORS.muted,
    fontSize: 12,
    lineHeight: 18,
  },
  moreText: {
    color: COLORS.green,
    fontSize: 12,
    fontWeight: '700',
    marginTop: 8,
  },
  center: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    paddingHorizontal: 28,
  },
  loadingText: {
    color: COLORS.muted,
    marginTop: 14,
  },
  errorIcon: {
    fontSize: 38,
    marginBottom: 12,
  },
  errorTitle: {
    color: COLORS.text,
    fontSize: 19,
    fontWeight: '800',
    marginBottom: 8,
  },
  errorText: {
    color: COLORS.muted,
    textAlign: 'center',
    lineHeight: 21,
  },
  retryButton: {
    marginTop: 20,
    paddingVertical: 12,
    paddingHorizontal: 18,
    backgroundColor: COLORS.green,
    borderRadius: 10,
  },
  retryText: {
    color: '#11170b',
    fontWeight: '800',
  },
  emptyText: {
    color: COLORS.muted,
    textAlign: 'center',
    marginTop: 50,
    paddingHorizontal: 20,
  },
  loadMoreButton: {
    alignItems: 'center',
    padding: 15,
    marginVertical: 10,
    backgroundColor: COLORS.card,
    borderRadius: 12,
  },
  loadMoreText: {
    color: COLORS.green,
    fontWeight: '800',
  },
  endText: {
    color: COLORS.muted,
    textAlign: 'center',
    padding: 18,
    fontSize: 12,
  },
  bottomBar: {
    borderTopWidth: 1,
    borderTopColor: COLORS.border,
    padding: 12,
    alignItems: 'center',
  },
  bottomText: {
    color: COLORS.muted,
    fontSize: 9,
    letterSpacing: 1.5,
  },
  backButton: {
    alignSelf: 'flex-start',
    paddingVertical: 12,
    marginBottom: 14,
  },
  backText: {
    color: COLORS.green,
    fontSize: 15,
    fontWeight: '700',
  },
  detailAvatar: {
    width: 190,
    height: 190,
    borderRadius: 22,
    alignSelf: 'center',
    marginVertical: 12,
  },
  detailIcon: {
    width: 150,
    height: 150,
    alignSelf: 'center',
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: COLORS.card,
    borderRadius: 24,
    marginVertical: 22,
  },
  detailEmoji: {
    fontSize: 68,
  },
  detailTitle: {
    color: COLORS.text,
    fontSize: 27,
    fontWeight: '900',
    textAlign: 'center',
    marginTop: 12,
  },
  detailSubtitle: {
    color: COLORS.green,
    textAlign: 'center',
    marginTop: 7,
    marginBottom: 24,
  },
  sectionTitle: {
    color: COLORS.text,
    fontSize: 18,
    fontWeight: '800',
    marginBottom: 12,
  },
  detailRow: {
    backgroundColor: COLORS.card,
    borderBottomWidth: 1,
    borderBottomColor: COLORS.border,
    padding: 14,
    borderRadius: 8,
    marginBottom: 5,
  },
  detailLabel: {
    color: COLORS.muted,
    fontSize: 12,
    marginBottom: 5,
  },
  detailValue: {
    color: COLORS.text,
    fontSize: 15,
    fontWeight: '600',
  },
  footerNote: {
    color: COLORS.muted,
    textAlign: 'center',
    fontSize: 11,
    marginVertical: 24,
  },
});
