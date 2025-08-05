import math
from collections import defaultdict
from tqdm import tqdm

def load_data():
    """Load and process book and rating data"""
    print("Loading book details...")
    isbn_to_title = {}
    book_id_to_isbn = {}
    
    # Read book data from Books.csv and map ISBNs to titles
    with open('Downloads/archive (1)/Books.csv', 'r', encoding='utf-8') as file:
        next(file)  # Skip the header row
        for idx, line in enumerate(file, start=1):
            try:
                parts = line.strip().split(';')
                isbn = parts[0].strip()
                title = parts[1].strip()
                isbn_to_title[isbn] = title
                book_id_to_isbn[idx] = isbn
            except Exception as error:
                print(f"Error reading line: {line}, Error: {error}")
                continue

    print("Loading user ratings from LIBSVM format...")
    user_ratings = defaultdict(dict)
    
    # Read user ratings from the ratings.libsvm file
    with open('Downloads/user_book_ratings_custom (2).libsvm', 'r') as file:
        for user_id, line in enumerate(file, start=1):
            ratings = line.strip().split()
            for rating in ratings:
                try:
                    if ':' not in rating:
                        continue
                    book_id, score = map(float, rating.split(':'))
                    user_ratings[user_id][int(book_id)] = score
                except ValueError as error:
                    print(f"Invalid rating skipped: {rating}, Error: {error}")
                    continue

    # Calculate user norms (used for cosine similarity)
    print("Calculating user norms...")
    user_norms = {
        user: math.sqrt(sum(rating ** 2 for rating in ratings.values()))
        for user, ratings in user_ratings.items()
        if sum(rating ** 2 for rating in ratings.values()) > 0
    }

    return user_ratings, isbn_to_title, user_norms, book_id_to_isbn

def compute_similarity(user1, user2, user_ratings, user_norms):
    """Calculate cosine similarity between two users"""
    if user1 not in user_norms or user2 not in user_norms:
        return 0.0

    shared_books = set(user_ratings[user1].keys()) & set(user_ratings[user2].keys())
    if not shared_books:
        return 0.0
    
    dot_product = sum(user_ratings[user1][book] * user_ratings[user2][book] for book in shared_books)
    denominator = user_norms[user1] * user_norms[user2]
    
    return dot_product / denominator if denominator > 0 else 0.0

def generate_recommendations(target_user, user_ratings, user_norms, k=10):
    """Recommend top 5 books for a user based on similar users"""
    similarity_scores = []
    
    for other_user in user_ratings:
        if other_user != target_user:
            sim = compute_similarity(target_user, other_user, user_ratings, user_norms)
            if sim > 0:
                similarity_scores.append((other_user, sim))
    
    top_similar_users = sorted(similarity_scores, key=lambda x: x[1], reverse=True)[:k]
    if not top_similar_users:
        return []
    
    target_user_books = set(user_ratings[target_user].keys())
    recommendations = {}
    
    for similar_user, similarity in top_similar_users:
        for book in user_ratings[similar_user]:
            if book not in target_user_books:
                numerator = sum(
                    user_ratings[sim_user][book] * sim for sim_user, sim in top_similar_users if book in user_ratings[sim_user]
                )
                denominator = sum(sim for _, sim in top_similar_users)
                if denominator > 0:
                    recommendations[book] = numerator / denominator

    return sorted(recommendations.items(), key=lambda x: x[1], reverse=True)[:5]

def main():
    """Main function to generate recommendations"""
    user_ratings, isbn_to_title, user_norms, book_id_to_isbn = load_data()
    
    print("Generating recommendations for all users...")
    total_users = len(user_ratings)
    
    with open('recommendationsFILE.csv', 'w', encoding='utf-8', newline='') as file:
        file.write('User_ID,Book_ID,Book_Title,Recommendation_Score\n')
        for user in tqdm(user_ratings.keys(), total=total_users, desc="Processing users"):
            recommendations = generate_recommendations(user, user_ratings, user_norms)
            for book_id, score in recommendations:
                title = isbn_to_title.get(book_id_to_isbn.get(book_id, str(book_id)), f"Book_{book_id}")
                scaled_score = min(max(round(score * 2), 1), 10)
                file.write(f'{user},{book_id},"{title}",{scaled_score}\n')

    print("Recommendations successfully generated!")

if _name_ == "_main_":
    main()
<img width="468" height="641" alt="image" src="https://github.com/user-attachments/assets/da2f7e88-0cc0-419d-a43b-b2066b68fc6d" />
